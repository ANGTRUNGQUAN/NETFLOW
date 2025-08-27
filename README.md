# NETFLOW
IPV4 AND IPV6
## Hướng dẫn Tăng kích thước FIFO (3 bước)
### Bước 1: Tạo IP Core FIFO Mới
Mở project của bạn trong Vivado hoặc ISE.

Tìm và khởi chạy công cụ "FIFO Generator".

Tạo một IP mới với các thông số cấu hình quan trọng sau:

Interface Type: Native (để có các tín hiệu cơ bản như din, dout, wr_en, rd_en).

FIFO Implementation: Independent Clocks Block RAM (hoặc Common Clock nếu bạn chắc chắn 100% xung nhịp ghi và đọc luôn giống hệt nhau).

Write Width: 72 (bits).

Write Depth: 1536 (Đây chính là giá trị 512 * 3).

Read Mode: Standard FIFO.

Read Width: 72 (bits).

Read Depth: 1536.

Hoàn tất quá trình tạo IP. Công cụ sẽ sinh ra một file template .v hoặc .xci cho bạn. Giả sử bạn đặt tên cho IP mới này là fifo_72x1536.

### Bước 2: Thay thế Mã nguồn trong file netflow_cache.v
Bây giờ, bạn sẽ xóa hoặc comment khối generate cũ và thay bằng một khối generate mới để khởi tạo IP fifo_72x1536 mà bạn vừa tạo.

Mã nguồn CŨ:
Verilog

// Khoi generate de tao cac instance FIFO (CŨ)
genvar L;
generate
    for (L = 0; L <= 3; L = L + 1) begin : export_fifo
        FIFO_SYNC_MACRO # (
            .DEVICE("VIRTEX5"),
            .DATA_WIDTH(72),
            .FIFO_SIZE("36Kb"), // <-- Độ sâu cũ tương đương 512
            ...
        ) FIFO_SYNC_MACRO_inst (
            .ALMOSTFULL(fifo_full_exp_int[L]),
            .DO(fifo_out_exp_int[L]),
            .EMPTY(fifo_empty_exp_int[L]),
            .CLK(ACLK),
            .DI(fifo_in_exp_int[L]),
            .RDEN(fifo_rd_exp_en),
            .RST(fifo_exp_rst),
            .WREN(fifo_w_exp_en)
        );
    end
endgenerate
Mã nguồn MỚI:
Bạn thay thế khối trên bằng khối dưới đây. Lưu ý tên cổng có thể hơi khác tùy theo phiên bản IP, bạn cần kiểm tra file template do IP Generator sinh ra.

Verilog

// Khoi generate de tao cac instance FIFO (MỚI với IP Generator)
genvar L;
generate
    for (L = 0; L <= 3; L = L + 1) begin : export_fifo_deep
        // Khởi tạo IP FIFO mới với độ sâu 1536
        fifo_72x1536 fifo_inst (
            .clk     (ACLK),               // Xung nhịp chung
            .rst     (fifo_exp_rst),       // Tín hiệu reset
            
            // Cổng Ghi (Write Port)
            .din     (fifo_in_exp_int[L]), // Dữ liệu vào 72-bit
            .wr_en   (fifo_w_exp_en),      // Cho phép ghi
            .full    (fifo_full_exp_int[L]), // Báo đầy
            
            // Cổng Đọc (Read Port)
            .dout    (fifo_out_exp_int[L]),// Dữ liệu ra 72-bit
            .rd_en   (fifo_rd_exp_en),     // Cho phép đọc
            .empty   (fifo_empty_exp_int[L]) // Báo rỗng
            
            // Các tín hiệu khác như wr_count, rd_count có thể nối nếu cần
        );
    end
endgenerate
### Bước 3: Sửa lại logic empty
Như đã phân tích ở lần trước, logic fifo_empty_exp của bạn đang bị sai. Hãy sửa nó lại để đảm bảo hệ thống hoạt động chính xác.

Verilog

// Logic noi cac tin hieu FIFO
// Sửa toán tử '|' thành '&' cho tín hiệu empty
assign fifo_empty_exp = fifo_empty_exp_int[3] & fifo_empty_exp_int[2] & fifo_empty_exp_int[1] & fifo_empty_exp_int[0];

// Logic cho full đã đúng, giữ nguyên
assign fifo_full_exp = fifo_full_exp_int[3] | fifo_full_exp_int[2] | fifo_full_exp_int[1] | fifo_full_exp_int[0];

// Logic ghép dữ liệu giữ nguyên
assign fifo_out_exp = {fifo_out_exp_int[3][23:0], fifo_out_exp_int[2], fifo_out_exp_int[1], fifo_out_exp_int[0]};
## Các Module Khác Có Cần Điều Chỉnh Không?
Câu trả lời là: Không. 🥳

Lý do là vì bạn đang thay đổi chi tiết triển khai (implementation detail) bên trong, chứ không thay đổi giao diện (interface).

Module export_expired_flows_from_mem chỉ quan tâm đến tín hiệu fifo_full_exp. Nó không cần biết FIFO sâu bao nhiêu. Nó chỉ biết rằng khi fifo_full_exp = 1, nó phải dừng lại. Việc tăng độ sâu chỉ làm cho tín hiệu này ít xuất hiện hơn.

Module axi4lite_flow_reader chỉ quan tâm đến fifo_empty_exp và fifo_out_exp. Nó cũng không cần biết độ sâu của FIFO. Nó chỉ biết rằng khi fifo_empty_exp = 0, nó có thể đọc dữ liệu.
