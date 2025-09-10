
# PBJ (345 điểm)

**Mô tả:** Con bot thực hiện tất cả các giao dịch, trong khi tôi ngồi không và thư giãn.
**Tác giả:** NoobMaster

## Phân tích ban đầu

<img width="697" height="194" alt="image" src="https://github.com/user-attachments/assets/14693030-971b-443d-9143-27af6af4cf43" />


Chúng ta được cung cấp mã nguồn của một hợp đồng thông minh Solidity, `Chall.sol`. Mục tiêu của thử thách là khai thác một lỗ hổng trong hợp đồng này để tăng số dư ETH của chúng ta lên hơn 50 ETH, được định nghĩa bởi hàm `isChallSolved()`.

Hợp đồng này triển khai một Nhà tạo lập thị trường tự động (AMM) đơn giản cho một token có tên là `flagCoin`. Hãy phân tích các thành phần chính của nó:

-   **Các biến trạng thái**:
    -   `flagCoin`: Tổng cung của token trong bể thanh khoản. Khởi tạo là 100.
    -   `eth`: Tổng lượng ETH trong bể thanh khoản.
    -   `k`: Tích số không đổi cho AMM, được tính bằng `eth * flagCoin`.
    -   `flags`: Một mapping để theo dõi số dư `flagCoin` của mỗi người dùng.

-   **Các hàm**:
    -   `constructor()`: Khởi tạo số dư `eth` bằng lượng ETH được gửi khi triển khai và tính toán tích số không đổi ban đầu `k`.
    -   `buy()`: Cho phép người dùng gửi ETH để mua `flagCoin`.
    -   `sell()`: Cho phép người dùng bán `flagCoin` của họ để nhận lại ETH.
    -   `isChallSolved()`: Kiểm tra xem số dư của người gọi có lớn hơn 50 ETH hay không.

Logic cốt lõi xoay quanh công thức tích số không đổi `eth * flagCoin = k`, đây là nền tảng của nhiều sàn giao dịch phi tập trung như Uniswap.

## Tìm kiếm lỗ hổng

Thoạt nhìn, logic của AMM có vẻ hợp lý. Các hàm `buy` và `sell` dường như cập nhật chính xác lượng dự trữ để duy trì hằng số bất biến. Tuy nhiên, một lỗ hổng rất tinh vi nhưng cực kỳ nghiêm trọng lại tồn tại.

Tích số không đổi `k` chỉ được tính một lần duy nhất trong hàm `constructor`:

```solidity
constructor() payable {
    eth = msg.value;
    k = eth * flagCoin;
}
```

Sau đó, giá trị của `k` **không bao giờ được cập nhật**. Cả hai hàm `buy()` và `sell()` đều sửa đổi lượng dự trữ `eth` và `flagCoin`, nhưng chúng không tính toán lại `k`. Hàm `sell()` dựa vào giá trị `k` cũ này để tính toán số tiền trả lại, đây chính là gốc rễ của lỗ hổng.

## Khai thác lỗ hổng

Việc khai thác bao gồm việc kết hợp vấn đề `k` cũ với bản chất của phép toán số nguyên trong Solidity.

1.  **Phép chia số nguyên trong `buy()`**:
    Số lượng `flagCoin` mà người dùng nhận được được tính như sau:
    ```solidity
    flag = (msg.value * flagCoin) / (eth + msg.value);
    ```
    Bởi vì phép chia của Solidity làm tròn xuống (floor), kết quả luôn được làm tròn xuống. Điều này có nghĩa là người dùng nhận được ít `flagCoin` hơn một chút so với giá trị toán học chính xác. Sai số làm tròn này mang lại lợi ích cho bể thanh khoản của hợp đồng.

2.  **Phá vỡ hằng số bất biến**:
    Bởi vì người dùng lấy ra ít coin hơn một chút so với mức họ nên nhận, tích số mới của lượng dự trữ (`new_eth * new_flagCoin`) trở nên **lớn hơn** một chút so với tích số ban đầu `k`. Trạng thái của hợp đồng bây giờ có một `k` "thực tế" cao hơn giá trị được lưu trong biến `k`.

3.  **Kiếm lợi từ `k` cũ trong `sell()`**:
    Khi chúng ta gọi hàm `sell()`, nó sẽ tính toán lượng ETH trả lại bằng cách sử dụng giá trị `k` ban đầu, đã cũ và nhỏ hơn.
    ```solidity
    // y là lượng dự trữ flagCoin mới, x là lượng dự trữ eth mới
    y = flag + flagCoin;
    x = k/y; // Sử dụng k cũ!
    to_pay = eth - x;
    ```
    Vì hợp đồng sử dụng một giá trị `k` nhỏ hơn hằng số bất biến "thực tế", nó tính ra một giá trị `x` nhỏ hơn (lượng dự trữ ETH mới). Điều này, đến lượt nó, dẫn đến một lượng `to_pay` **lớn hơn** cho người bán (`to_pay = current_eth - smaller_x`).

    Lợi nhuận từ mỗi chu kỳ chính xác là `(k_actual - k_stale) / y`.

## Giải pháp

Chiến lược là thực hiện một chu kỳ `mua-rồi-bán` lặp đi lặp lại để tích lũy lợi nhuận nhỏ từ mỗi giao dịch.

1.  **Mua**: Gọi hàm `buy()` với một lượng ETH đáng kể. Kích thước giao dịch lớn hơn có thể dẫn đến sai số làm tròn lớn hơn, làm tăng `k_actual` và do đó tăng lợi nhuận tiềm năng.
2.  **Bán**: Ngay lập tức gọi hàm `sell()` để bán tất cả `flagCoin` đã mua ở bước trước.
3.  **Lặp lại**: Lặp lại quá trình này. Với mỗi chu kỳ, số dư ETH của chúng ta tăng lên một chút.

Script giải quyết được cung cấp, `pbj_solver_web3.py`, tự động hóa chính xác quá trình này. Nó đi vào một vòng lặp, mua `flagCoin` với một phần lớn số dư của mình và sau đó bán ngay lập tức. Nó kiểm tra lợi nhuận và tiếp tục cho đến khi `isChallSolved()` trả về true.

```python
# Đoạn mã từ script giải quyết minh họa vòng lặp
while get_flagcoin_total() > 0 and get_balance() > 2:
    eth_balance_before = get_balance()
    # ... tính toán buy_amount ...
    buy_flagcoin(buy_amount)
    time.sleep(2)
    flagcoin = get_flagcoin()
    if flagcoin > 0:
        sell_flagcoin(flagcoin)
        time.sleep(2)
        eth_balance_after = get_balance()
        if eth_balance_after > eth_balance_before:
            print("Bán có lãi, tiếp tục vòng lặp.")
            if is_solved():
                print("Challenge solved!")
                get_flag(Secret)
                break
        # ...
```

Bằng cách chạy script này, số dư của chúng ta cuối cùng sẽ vượt qua 50 ETH và chúng ta có thể lấy được flag.

### Cách khắc phục lỗ hổng

Việc sửa lỗi rất đơn giản: hợp đồng nên tính toán lại `k` sau mỗi giao dịch làm thay đổi lượng dự trữ. Thêm `k = eth * flagCoin;` vào cuối cả hai hàm `buy()` và `sell()` sẽ vá được lỗ hổng này.

