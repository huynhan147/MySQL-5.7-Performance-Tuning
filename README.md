[Source](https://www.percona.com/blog/2016/10/12/mysql-5-7-performance-tuning-immediately-after-installation/ "Permalink to MySQL 5.7 Performance Tuning Immediately After Installation")

# Chỉnh sửa hiệu suất MySQL 5.7 ngay lập tức sau khi cài đặt

Blog này cập nhật [ blog của Stephane Combaudon về điều chỉnh hiệu suất MySQL ][1] và bao gồm điều chỉnh hiệu suất MySQL 5.7 ngay lập tức sau khi cài đặt.

Một vài năm trước, Stephane Combaudon đã viết một bài đăng trên blog về [ 10 cài đặt điều chỉnh hiệu năng MySQL sau khi cài đặt ][1] bao gồm các phiên bản cũ hơn (hiện tại) của MySQL: 5.1, 5.5 và 5.6. Trong bài này, tôi sẽ xem xét những gì để điều chỉnh trong MySQL 5.7 (tập trung vào InnoDB).

Tin tốt là MySQL 5.7 có giá trị mặc định tốt hơn đáng kể. Morgan Tocker đã tạo một [trang có danh sách đầy đủ các tính năng trong MySQL 5.7] [2], và là một điểm tham khảo tuyệt vời. Ví dụ: các biến sau được đặt _mặc định_:

- innodb_file_per_table=ON
- innodb_stats_on_metadata = OFF
- innodb_buffer_pool_instances = 8 (hoặc 1 nếu innodb_buffer_pool_size < 1GB)
- query_cache_type = 0; query_cache_size = 0; (vô hiệu hóa mutex)

Trong MySQL 5.7, chỉ có bốn biến thực sự quan trọng cần được thay đổi. Tuy nhiên, có các biến khác của InnoDB và MySQL toàn cục có thể cần được điều chỉnh cho một khối lượng công việc và phần cứng cụ thể.

Để bắt đầu, hãy thêm các cài đặt sau vào my.cnf trong phần [mysqld]. Bạn sẽ cần phải khởi động lại MySQL:

```[mysqld]
 # other variables here
  innodb_buffer_pool_size = 1G # (adjust value here, 50%-70% of total RAM) 
  innodb_log_file_size = 256M 
  innodb_flush_log_at_trx_commit = 1 # may change to 2 or 0 innodb_flush_method = O_DIRECT
```

| ----- |
|   | 

[mysqld]

# other variables here

innodb_buffer_pool_size = 1G # (adjust value here, 50%-70% of total RAM)

innodb_log_file_size = 256M

innodb_flush_log_at_trx_commit = 1 # may change to 2 or 0

innodb_flush_method = O_DIRECT

 | 

Mô tả :

| ----- |
| **Biến** |  **Giá trị** |  
| innodb_buffer_pool_size |  Bắt đầu với 50% 70% tổng số RAM. Không cần phải lớn hơn kích thước cơ sở dữ liệu |  
| innodb_flush_log_at_trx_commit | 

* 1   (Mặc định)
* 0/2 (hiệu suất cao hơn, độ tin cậy thấp hơn)
 |  
| innodb_log_file_size |  128M – 2G (không cần phải lớn hơn vùng đệm) |  
| innodb_flush_method |  O_DIRECT (tránh 2 lần buffering) | 

 

_**Tiếp theo là gì?**_

Đó là một điểm khởi đầu tốt cho bất kỳ cài đặt mới nào. Có một số biến khác có thể tăng hiệu năng MySQL cho một số tải công việc. Thông thường, tôi sẽ thiết lập một công cụ theo dõi / vẽ đồ thị MySQL (ví dụ, [nền tảng giám sát và quản lý Percona] [3]) và sau đó kiểm tra bảng điều khiển MySQL để thực hiện điều chỉnh thêm.

_**Những gì chúng ta có thể điều chỉnh thêm dựa trên các đồ thị?**_

_Kích thước vùng đệm của InnoDB_. Xem đồ thị sau:

![MySQL 5.7 Performance Tuning][4]

![MySQL 5.7 Performance Tuning][5]

Như chúng ta có thể thấy, chúng ta có thể có lợi từ việc tăng kích thước vùng đệm InnoDB một chút lên ~ 10G, vì chúng ta có RAM và số trang free nhỏ so với tổng số vùng đệm.

_Kích thước file log InnoDB._ Xem đồ thị sau:

![MySQL 5.7 Performance Tuning][6]
	
Như chúng ta có thể thấy ở đây, InnoDB thường ghi 2,26 GB dữ liệu mỗi giờ, vượt quá tổng kích thước của các file log (2G). Bây giờ chúng ta có thể tăng biến innodb_log_file_size và khởi động lại MySQL. Ngoài ra, hãy sử dụng "show engine InnoDB status" để [tính toán kích thước file log InnoDB tốt] [7].

_**Các biến khác**_

Có một số biến InnoDB khác có thể được điều chỉnh thêm:

_innodb_autoinc_lock_mode_

Thiết lập [innodb_autoinc_lock_mode] [8] = 2 (chế độ xen kẽ) có thể loại bỏ nhu cầu cho khóa AUTO-INC cấp bảng (và có thể tăng hiệu suất khi các câu lệnh insert nhiều hàng được sử dụng để thêm các giá trị vào các bảng với khóa chính auto_increment). Điều này đòi hỏi binlog_format = ROW hoặc MIXED (và ROW là mặc định trong MySQL 5.7).

_innodb_io_capacity _and_ innodb_io_capacity_max_

Đây là một điều chỉnh nâng cao hơn, và chỉ có ý nghĩa khi bạn đang thực hiện việc viết quá nhiều (nó không áp dụng cho lần đọc, tức là SELECT). Nếu bạn thực sự cần phải điều chỉnh nó, phương pháp tốt nhất là biết bao nhiêu IOPS hệ thống có thể thực hiện. Ví dụ, nếu máy chủ có một ổ SSD, chúng ta có thể thiết lập innodb_io_capacity_max = 6000 và innodb_io_capacity = 3000 (50% tối đa). Đó là một ý tưởng tốt để chạy sysbench hoặc bất kỳ công cụ benchmark khác nào để chuẩn hóa thông lượng đĩa.


Nhưng chúng ta có cần phải lo lắng về cài đặt này không? Xem biểu đồ về "[trang bẩn] [9]" của vùng đệm :

![screen-shot-2016-10-03-at-7-19-47-pm][10]

Trong trường hợp này, tổng số trang bẩn là cao, và có vẻ như InnoDB không thể theo kịp với flush chúng. Nếu chúng tôi có hệ thống phụ đĩa nhanh (nghĩa là SSD), chúng tôi có thể hưởng lợi từ việc tăng innodb_io_capacity và innodb_io_capacity_max.

_**Kết luận hoặc TL; DR phiên bản**_

Các mặc định mới của MySQL 5.7 tốt hơn nhiều cho những khối lượng công việc với mục đích chung. Đồng thời, chúng ta vẫn cần cấu hình biến InnoDB để tận dụng số lượng RAM trên hộp. Sau khi cài đặt, hãy làm theo các bước sau:

1. Thêm biến InnoDB vào my.cnf (như mô tả ở trên) và khởi động lại MySQL
2. Cài đặt hệ thống giám sát, (ví dụ: nền tảng giám sát và quản lý Percona)
3. Nhìn vào biểu đồ và xác định xem liệu MySQL có cần được điều chỉnh thêm hay không
