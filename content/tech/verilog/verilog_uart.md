---
title: "UART串口"
date: 2026-06-24T09:04:38.710Z
categories:
  - verilog
tags:
  - verilog
cover: "https://xuejian-1432996375.cos.ap-beijing.myqcloud.com/科技/verilog/uart/clipboard_20260624_045912.png"
---

串口分为接收和发送两个协议分别是TX和RX。
![图片](https://xuejian-1432996375.cos.ap-beijing.myqcloud.com/科技/verilog/uart/clipboard_20260624_045912.png)
串口协议图如下图所示：
![图片](https://xuejian-1432996375.cos.ap-beijing.myqcloud.com/科技/verilog/uart/clipboard_20260624_045924.png)
其中最开始为起始位，中间是数据位，然后奇偶校验区间，最后是终止位。
起始位规定为低电平，终止位规定为高电平。中间8位为数据位。奇偶校验位可以根据选择添加或者不添加。就是状态机多个状态，校验状态。
奇校验（Odd Parity）： 让数据位+校验位里1的总数为奇数。
偶校验（Even Parity）： 让数据位+校验位里1的总数为偶数。
接收方收到数据后自己算一遍，如果1的个数不对说明传输出错了。
RX采样选择在中间采样，这样避免毛刺导致错误的开始接收。
TX时序图如下图所示：
![图片](https://xuejian-1432996375.cos.ap-beijing.myqcloud.com/科技/verilog/uart/clipboard_20260624_045927.png)
RX时序图如下图所示
![图片](https://xuejian-1432996375.cos.ap-beijing.myqcloud.com/科技/verilog/uart/clipboard_20260624_045930.png)
RX状态机变化图如下图所示
![图片](https://xuejian-1432996375.cos.ap-beijing.myqcloud.com/科技/verilog/uart/clipboard_20260624_045933.png)
代码如下所示：
波特率计算公式
baud_count = clk_freq / baud_rate - 1
**TX代码如下所示：**
module uart_tx#(
    parameter clk_freq = 50000000, //50MHz
    parameter baud_rate = 9600
)(
    input clk,
    input rst_n,
    input [7:0] data_in,
    input send,
    output reg tx,
    output reg busy
);
    localparam IDLE = 2'b00;
    localparam DATA = 2'b01;
    localparam STOP = 2'b10;
    localparam baud_count = clk_freq/baud_rate-1;
    
    reg [1:0] state;
    reg [3:0] bit_count;//记录当前在传输第几位
    reg [12:0] period_count;
    reg [7:0] data_in_reg; //输入数据寄存
    //状态机变换
    always@(posedge clk or negedge rst_n) begin
        if(!rst_n) begin
            state <= IDLE;
        end
        else begin
            case(state)
                IDLE:state <= (send & (!busy))?DATA:IDLE; //判断是否空闲和请求发送
                DATA:state <= (bit_count == 8'd8 && (period_count == baud_count))?STOP:DATA; //判断是否在停止位
                STOP:state <= (period_count == baud_count)?IDLE:STOP; //判断是否结束传输
                default:state <= IDLE; 
            endcase
        end
    end 
    //数据传输
    always@(posedge clk or negedge rst_n) begin
        if(!rst_n) begin
            tx<=1;
            busy <= 0;
            bit_count <= 0;
            period_count <= 0;
        end
        else 
            case(state)
                IDLE:begin
                    tx <= 1;
                    busy <= 0;
                    bit_count <= 0;
                    period_count <= 0;
                    data_in_reg <= data_in;
                end
                DATA:begin
                    busy <= 1;
                        if(period_count == baud_count)begin
                            bit_count <= bit_count +1;
                            period_count <= 0;
                        end
                        else begin
                            period_count <= period_count +1;
                            if(bit_count == 0)
                                tx <= 0;
                            else
                                tx <= data_in_reg[bit_count-1];
                        end
                end
                STOP:begin
                    busy <= 1;
                    tx <= 1;
                    if(period_count == baud_count)
                        period_count <= 0;
                    else
                        period_count <= period_count + 1;
                end
                default:begin
                    tx <= 1;
                    busy <= 0;
                end
            endcase
    end
endmodule

**RX代码如下所示：**
module uart_rx#(
    parameter clk_freq = 50000000, //50MHz
    parameter baud_rate = 9600
)(
    input             clk,
    input             rst_n,
    input             rx,
    output reg        valid,
    output reg  [7:0] data_out
);
    localparam baud_count = clk_freq/baud_rate-1; //波特率标签位
    localparam baud_count_mid= clk_freq/(2*baud_rate)-1; //中间标签位
    localparam IDLE = 2'b00; 
    localparam START = 2'b01;
    localparam DATA = 2'b10;
    localparam STOP = 2'b11;
    reg [3:0] bit_count;
    reg [12:0] period_count;
    reg [1:0] state;


    //状态机状态变化
    always@(posedge clk or negedge rst_n) begin
        if(!rst_n)begin
            state <= IDLE;
        end
        else
            case(state)
                IDLE:state <= (!rx)?START:IDLE;
                START:state <= (period_count == baud_count_mid)?((!rx)?DATA:IDLE):START;
                DATA:state <= ((bit_count == 4'd8) && (period_count == baud_count))?STOP:DATA;
                STOP:state <= (period_count == baud_count)?IDLE:STOP;
                default:state <= IDLE;
            endcase
    end
    always@(posedge clk or negedge rst_n) begin
        if(!rst_n) begin
            period_count <= 0;
            bit_count <= 0;
            valid <= 0;
            data_out <= 8'd0;
        end
        else
            case(state)
                IDLE:begin
                    bit_count <= 0;
                    valid <= 0;
                    data_out <= 8'd0;
                    period_count <= 4'd0;
                end
                START:begin
                        period_count <= period_count + 1;
                end
                DATA:begin
                    if(period_count == baud_count_mid)
                        data_out[bit_count-1] <= rx;
                    if(period_count == baud_count) begin
                        bit_count <= bit_count + 1;
                        period_count <= 0;
                    end 
                    else 
                        period_count <= period_count + 1;
                end
                STOP:begin
                    if(period_count == baud_count) begin
                        bit_count <= bit_count + 1;
                        period_count <= 0;
                        valid <= 1;
                    end 
                    else begin
                        period_count <= period_count + 1;
                        valid <= 0;
                    end
                        
                end
            endcase
    end
    
endmodule
补充：
START跳DATA时period_count不清零， 让采样点自然落在每位中间。 data_out[bit_count-1]的-1是配套的索引补偿。
鲁棒性不足的地方：
没有帧错误检测 没有超时保护 data_out没有影子寄存器

