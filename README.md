# 32-bit Pipelined ALU

本專案以硬體流水線（Pipeline）架構，從零設計一顆 32-bit 運算單元（ALU），完整涵蓋 RTL 撰寫、功能驗證、Synthesis 與 STA 時序分析，並透過架構優化驗證「架構設計決定電路效能上限」的核心觀點。

## 開發進度與學習日誌

- [點我觀看專題製作過程與 Debug 日誌](./32-bit_ALU_log.md) — 完整記錄從 Baseline 到 Pipeline 的架構決策與除錯過程
- [點我觀看每日學習與 Debug 日誌](./learning_log.md) — 從 Verilog 語法自學到 Vivado 工具鏈的每日進度
- [點我觀看參考文獻](./References.md)

---

## 核心亮點與最終時序對比

針對 Baseline（純組合邏輯）與 Pipeline（2-Stage）版本，使用相同的 10 筆測試向量驗證功能等價後，以逐步壓低 clock period 的方式收斂至 WNS 接近 0，取得實際可運作頻率上限：

| 項目                          | Baseline (alu_v1)      | Pipeline (alu_v2)          |
|-------------------------------|------------------------|----------------------------|
| Period（收斂值）              | 6.3 ns                 | 3.65 ns                    |
| WNS                           | 0.064 ns               | 0.075 ns                   |
| Data Path Delay（關鍵路徑）   | 6.245 ns               | 3.515 ns                   |
| Fmax                          | ≈ 160.35 MHz           | ≈ 279.72 MHz (**+74.5%**)  |
| LUT                           | 349                    | 323                        |
| FF                            | 103                    | 240                        |
| Latency                       | 1 cycle                | 2 cycle                    |

### 核心要點

1. **Baseline** 為純組合邏輯（外包 wrapper 以建立 register-to-register path）。加減法以 + 描述，Vivado 對應到 CARRY4 進位鏈。Path 1 由 opcode → 進位鏈 → 結果選擇 → Zero Flag 串接而成，Data Path Delay 6.245 ns，Fmax ≈ 160.35 MHz。

2. **Pipeline** 將資料路徑切成 2 個 stage，中間插入暫存器，單一 stage 的關鍵路徑延遲由 6.245 ns 降至 3.515 ns，Fmax 提升至約 279.72 MHz（約 1.75 倍）。

3. **Trade-off**
   - 用暫存器（FF）數量換取頻率與吞吐量：FF 從 103 → 240。
   - 因 Stage1 已算出 add_sub 結果，Stage2 選擇邏輯可直接沿用，Synthesizer 得以合併重複的輸出多工器，LUT 用量反而略降（349 → 323）。
   - Latency 從 1 cycle 增加為 2 cycle（指輸入暫存器到輸出暫存器之間的運算週期數），犧牲單筆資料延遲，換取整體吞吐量與可運作頻率的提升。

4. 兩個版本皆使用同一組 10 筆測試向量（V2 因 pipeline 需等待 2 個 clock 後取樣）（含 ADD/SUB、邏輯運算、SRA 符號延伸、Overflow 邊界、有號數 SLT、shamt 極值 0/31）驗證，確保時序比較建立在功能正確的公平基準上。

---

## 架構設計與流水線切割策略
<img width="544" height="316" alt="image" src="https://github.com/user-attachments/assets/edfdfcaf-8039-4a88-8e76-640d7a6a100d" />
<img width="923" height="507" alt="image" src="https://github.com/user-attachments/assets/24a57abe-c425-42a0-8ac1-af8253aafec1" />

---

### alu_v1 程式碼
#### 1. alu_v1_core
```verilog
module alu_v1(
    input [31:0]a,
    input [31:0]b,
    input [2:0]op_code,
    output reg[31:0]res,
    output [3:0]flag
);
    //構建4個 32bit 暫存器用於 barrel shifter
    //Shamt 選用 b[4:0] 判斷移動幾個 bit
    reg [31:0]t1, t2, t3, t4;
    
    // 構建共用的 33-bit 加減法邏輯（在 case 外面算）
    wire sub;
    wire [31:0]b_op;
    wire [32:0]add_sub;
    wire carry;
    wire overflow;

    assign sub = (op_code == 3'b001);// op_code == 3'b001，執行減法
    assign b_op = b ^ {32{sub}}; // 將 b 逐位元反相
    assign add_sub = {1'b0, a} + {1'b0, b_op} + sub; // ADD: a+b；SUB: a + ~b + 1（2補數，+1 由 sub 提供）
    assign carry = add_sub[32]; //進位判斷 
    assign overflow = (~(a[31] ^ b_op[31])) && (a[31] ^ add_sub[31]);//溢位判斷

// 判斷 ALU 行為
always@(*)begin
    case(op_code)
        3'b000 : // a+b
            res = add_sub[31:0]; 
        3'b001 : // a-b 
            res = add_sub[31:0];
        3'b010 : 
            res = a & b;
        3'b011 : 
            res = a | b;
        3'b100 : 
            res = a ^ b;
        3'b101 : begin // 邏輯左移(<<)，缺項補0
            t1 = (b[0]) ? {a[30:0], 1'b0} : a;
            t2 = (b[1]) ? {t1[29:0], 2'b00} : t1;
            t3 = (b[2]) ? {t2[27:0], 4'b0000} : t2;
            t4 = (b[3]) ? {t3[23:0], 8'b00000000} : t3;
            res = (b[4]) ? {t4[15:0], 16'b00000000} : t4;
        end 
        3'b110 : begin // 算術右移(>>>)，補 signed bit a[31]
            t1 = (b[0]) ? {a[31], a[31:1]} : a;
            t2 = (b[1]) ? {{2{a[31]}}, t1[31:2]} : t1;
            t3 = (b[2]) ? {{4{a[31]}}, t2[31:4]} : t2;
            t4 = (b[3]) ? {{8{a[31]}}, t3[31:8]} : t3;
            res = (b[4]) ? {{16{a[31]}}, t4[31:16]} : t4; 
        end
        3'b111 : res = ($signed(a) < $signed(b)) ? 32'd1 : 32'd0; // 有號數比較
        default : res = 32'd0;
    endcase
end

// 構建 flag
wire a_s;
wire Z, N, C, V;

// flag 只在 ADD/SUB 時才有意義
assign a_s = (op_code == 3'b000) || (op_code == 3'b001);

assign Z = (res == 32'd0) ? 1 : 0; // 零旗標判斷
assign N = (res[31]) ? 1 : 0; // 負旗標判斷
assign C = (a_s) ? carry : 0; // 進位旗標判斷
assign V = (a_s) ? overflow : 0; // 溢位旗標判斷
    
assign flag = {Z, N, C, V}; // flag[3]=Z, flag[2]=N, flag[1]=C, flag[0]=V

endmodule
```

#### 2. alu_v1_tt
```verilog
module alu_v1_tt();

    reg [31:0]a, b;
    reg [2:0]op_code;
    wire [31:0]res;
    wire [3:0]flag;
    integer i;
    
    integer error_count; // 用於計算"error"次數
    integer test_count; // 用於計算"test"次數
    
    alu_v1 tt(
    .a(a),
    .b(b),
    .op_code(op_code),
    .res(res),
    .flag(flag)
    );
    
    task automatic check(
        input [31:0]t_a, // a的測試變數
        input [31:0]t_b, // b的測試變數
        input [2:0]t_op, // op_code的測試變數
        input [31:0]t_res, // res的測試變數
        input t_z, t_n, t_c, t_v // flag的測試變數
    );
        begin
            a = t_a; b = t_b; op_code = t_op;
            #10;
            test_count = test_count + 1; // 每次開始測試 test_count 次數 +1
            if(res !== t_res || flag !== {t_z, t_n, t_c, t_v})begin // res 結果或 flag 結果不符
                error_count = error_count + 1; // 錯誤就把 error_count 次數 +1
                $display("[FAIL]：%0d, op_code = %b, a = %h, b = %h | got res = %h, flag = %b | t_res = %h, flag = %b",
                test_count, t_op, t_a, t_b, res, flag, t_res, {t_z, t_n, t_c, t_v});
            end
            else begin
                $display("[PASS]：%0d, op_code = %b", test_count, t_op);
            end
        end
    endtask
    
    initial begin
        error_count = 0;
        test_count = 0;
        
        // 基本功能：ADD / SUB 
        check(32'h0000_00F0, 32'h0000_000F, 3'b000, 32'h0000_00FF, 0,0,0,0); // ADD
        check(32'h0000_00F0, 32'h0000_000F, 3'b001, 32'h0000_00E1, 0,0,1,0); // SUB
        
        // 邏輯運算：AND / OR / XOR
        check(32'hFF00_FF00, 32'h0F0F_0F0F, 3'b010, 32'h0F00_0F00, 0,0,0,0); // AND
        check(32'hFF00_FF00, 32'h0F0F_0F0F, 3'b011, 32'hFF0F_FF0F, 0,1,0,0); // OR
        check(32'hFF00_FF00, 32'h0F0F_0F0F, 3'b100, 32'hF00F_F00F, 0,1,0,0); // XOR
        
        // 針對電路容易出錯的地方各自設計測資
        check(32'hF123_4567, 32'h0000_0004, 3'b110, 32'hFF12_3456, 0,1,0,0); // SRA 符號延伸
        check(32'h7FFF_FFFF, 32'h0000_0001, 3'b000, 32'h8000_0000, 0,1,0,1); // ADD overflow
        check(32'hFFFF_FFFF, 32'h0000_0005, 3'b111, 32'h0000_0001, 0,0,0,0); // SLT：-1 < 5
        check(32'h0000_0001, 32'h0000_0000, 3'b101, 32'h0000_0001, 0,0,0,0); // shamt=0
        check(32'h0000_0001, 32'h0000_001F, 3'b101, 32'h8000_0000, 0,1,0,0); // shamt=31
        
        $display("\n測試完成：共 %0d 組，失敗 %0d 組", test_count, error_count);
        if(error_count == 0)begin
            $display("ALL TESTS PASSED");
        end
        $finish;       
    end  
endmodule
```

#### 3. alu_v1_wrapper
```verilog
module alu_v1_wrapper(
    input clk,
    input [31:0]a_in,
    input [31:0]b_in,
    input [2:0]op_in,
    output reg [31:0]res_out,
    output reg  [3:0]flag_out
);
    
    reg [31:0] a_r; // 鎖住輸入的暫存器，posedge clk 時取樣 a_in/b_in
    reg [31:0] b_r;
    reg [2:0] op_r; // 鎖住輸入的暫存器，posedge clk 時取樣 op_in
    wire [31:0] res_w; // alu_v1 算出的組合邏輯結果（尚未鎖存）
    wire [3:0] flag_w; // alu_v1 算出的組合邏輯 flag（尚未鎖存）
    
    always@(posedge clk)begin
        /*
        輸入暫存器：把當下的 a_in/b_in/op_in 取樣鎖住，讓 alu_v1 core 拿到的是穩定值
        ，而不是隨時可能變動的外部輸入
        */
        a_r <= a_in; 
        b_r <= b_in;
        op_r <= op_in;
        // 輸出暫存器：把 alu_v1 算好的組合邏輯結果取樣鎖住，才能輸出穩定的 res_out/flag_out
        res_out <= res_w;
        flag_out <= flag_w;
    end
    
    // alu_v1：純組合邏輯核心，接的是已經被鎖住的 a_r/b_r/op_r（不是原始輸入a_in、b_in、op_in）
    alu_v1 core(
    .a(a_r),
    .b(b_r),
    .op_code(op_r),
    .res(res_w),
    .flag(flag_w) 
    );

endmodule
```

---
### 在 alu_v1 中的 critical path
<img width="1803" height="487" alt="image" src="https://github.com/user-attachments/assets/94f08a07-f7a5-4252-8ef5-b7cab7067b97" />
<img width="512" height="380" alt="image" src="https://github.com/user-attachments/assets/9f630dd6-cf42-44b0-a244-63a83d307b14" />

依據 Vivado Timing Report（Path 1，Slack 0.064 ns）：

- 起點 → 終點：`op_r_reg[0]/C` → `flag_out_reg[3]/D`（Zero Flag）
- Data Path Delay = 6.245 ns（Logic 2.607 ns, 41.7%｜Route 3.638 ns, 58.3%）
- Logic Levels = 11（CARRY4 × 6、LUT4 × 2、LUT6 × 3）

#### 最主要的 Critical Path：Opcode → 加減法進位鏈 → 結果選擇 → Zero Flag

`op_r_reg[0]`（fanout = 122）先產生 `sub`（LUT4，同時作為加減法器的進位輸入），經 6 級 CARRY4 進位鏈，再經 2 級 LUT6（`res_out[23]` 的結果選擇），最後由 LUT4 + LUT6 做 32-bit Zero Flag 縮減並鎖入 `flag_out_reg[3]`。

| 階段 | 延遲 (ns) | 佔比 |
|---|---|---|
| 起點暫存器 clk→Q + 高扇出（fo = 122）繞線 | 1.181 | 18.9% |
| LUT4（sub）+ 6 級 CARRY4 進位鏈 | 1.668 | 26.7% |
| 加法器輸出 → 結果選擇（2 級 LUT6 + 繞線） | 2.468 | 39.5% |
| Zero Flag 縮減（LUT4 + LUT6） | 0.928 | 14.9% |
| 合計 | 6.245 | 100% |

觀察：
1. 進位鏈本體僅約 1.35 ns（21.6%）。`+` 由 Vivado 對應到 FPGA 專用的 CARRY4 進位鏈，而非 32 個 full adder 逐級串接。
2. 以加法器輸出為界，前段（起點 → 進位鏈）約 2.849 ns、後段（結果選擇 + Zero Flag）約 3.396 ns，這是 pipeline 選擇在 add_sub 輸出處切割的依據。
3. Route 佔 58.3%，且起點 `op_r_reg[0]` fanout 高達 122（單一 net 0.725 ns），顯示高扇出控制訊號在 baseline 就已影響時序。

#### 次要 Critical Path：Barrel Shifter

Top-10 中其餘 6 條路徑起點為 `b_r_reg` / `a_r_reg`、終點為 `res_out_reg`（4 級邏輯、Total Delay 4.918 ~ 5.478 ns、Slack 0.753 ~ 1.290 ns），對應 5 階 MUX 的移位器結構。其 Total Delay 遠高於 V2 的 3.515 ns，因此移位器也必須一併切割。

#### 位元邏輯運算（AND / OR / XOR）：非瓶頸
  * 邏輯結構：32-bit 平行運算。
  * 延遲原因：僅需經過 1 階邏輯門，無位元間的依賴關係，延遲極短。

#### SLT（Set Less Than）：非瓶頸
  * SLT 在 Baseline 中主要透過有號數比較實現，延遲相對較短，並非關鍵路徑瓶頸。
---

### 2-Stage Pipeline 切割原則
<img width="581" height="648" alt="image" src="https://github.com/user-attachments/assets/084ded80-5b4e-479d-ac07-4312b5f50bf4" />

#### 2-Stage Pipeline 切割方案 (縱向階段切割)
1. Stage 1：Shifter 前 3 階 (b[0]~b[2]: 1, 2, 4 bits 移位) ； 33-bit 加減法運算 (ADD / SUB / SLT)
2. Stage 2：Shifter 後 2 階 (b[3]~b[4]: 8, 16 bits 移位)；Flag 生成 (Z, N, C, V)

#### 2-Stage Pipeline 切割方案原因
1. 時間延遲的「均等平分」（Timing Balance）：Pipeline 切割遵循『各級延遲盡量平衡』原則（balanced pipeline staging），避免單一 stage 成為新的瓶頸，此為流水線設計的基本準則（參考 Hennessy & Patterson, Computer Organization and Design, Pipelining 章節）。
    * 桶型移位器總共有 5 階 MUX 選擇器
    * Stage 1 切前 3 階（1, 2, 4 bits）： 穿過 3 個 MUX。
    * Stage 2 切後 2 階（8, 16 bits）： 穿過 2 個 MUX ＋ 1 個最終輸出的 MUX（多路選擇 case）。
    * 讓 Stage 1 與 Stage 2 的邏輯門延遲（Gate Delay）平分
    
2. 切點選在加法器輸出
    * Baseline Path 1 以 add_sub 輸出為界，前段約 2.849 ns、後段（結果選擇 + Zero Flag）約 3.396 ns，兩段長度接近。
    * 因此將 add_sub 結果鎖進 Stage 1 暫存器，並讓移位器前 3 階（1, 2, 4 bits）與加減法器並行放在 Stage 1。
        
3. 簡化 Stage 2 的 SLT 與 Flag 計算
	* 因為加法器在 Stage 1 已經算完了 add_sub ，所以 Stage 2 就可以直接利用 Stage 1 留下來的 add / sub 結果來判斷溢位（Overflow）與產生 Flag，不會擠壓到 Stage 2 的時序。

4. Pipeline 化後 Zero Flag 不再是最長路徑
    * Baseline 中，Zero Flag 疊加在加法器與結果選擇之後（Zero Flag 縮減約 0.928 ns）。
    * V2 的 Stage 2 不再包含加法器。V2 最長路徑（Path 1）變為 `op_code_stg1_reg[0]` → `res_reg[2]`（結果選擇的控制訊號，3 級邏輯、fanout = 33），Flag 路徑最差為 Path 5（Slack 0.187 ns）。

5. Timing Balance（時序平衡）
    * 實測 Stage 1 最差 slack（`b_r_reg[0]/C` → `core/res_stg1_reg[0]/D`）Total Delay = 3.441 ns（Slack 0.164 ns）；Stage 2 最差 slack（`core/op_code_stg1_reg[0]/C` → `core/res_reg[2]/D`）Total Delay = 3.515 ns（Slack 0.075 ns）。兩級差距約 0.074 ns（約 2%），大致平衡。
---

### alu_v2 程式碼
#### 1. alu_v2_core
```verilog
module alu_v2(
    input clk,
    input rst_n,
    input [31:0]a,
    input [31:0]b,
    input [2:0]op_code,
    output reg[31:0]res,
    output reg[3:0]flag
);

// --- stag1：前半段運算　＋　暫存器（stag1 registers）---

    //構建3個 32bit 暫存器用於 barrel shifter
    //Shamt 選用 b[0] ~ b[2] 判斷移動幾個 bit
    wire [31:0]s1_l, s2_l, s3_l; 
    wire [31:0]s1_r, s2_r, s3_r;
    
    // 構建共用的 33-bit 加減法邏輯（在 case 外面算）
    wire sub;
    wire [31:0]b_op;
    wire [32:0]add_sub;

    assign sub = (op_code == 3'b001); // op_code == 3'b001，執行減法
    assign b_op = b ^ {32{sub}}; // 將 b 逐位元反相
    assign add_sub = {1'b0, a} + {1'b0, b_op} + sub; // ADD: a+b；SUB: a + ~b + 1（2補數，+1 由 sub 提供）
    
    // b[0] ~ b[2] 的 logic_shift_left
    assign s1_l = (b[0]) ? {a[30:0], 1'b0} : a;
    assign s2_l = (b[1]) ? {s1_l[29:0], 2'b00} : s1_l;
    assign s3_l = (b[2]) ? {s2_l[27:0], 4'b0000} : s2_l;
    
    // b[0] ~ b[2] 的 arithmetic_shift_right
    assign s1_r = (b[0]) ? {a[31], a[31:1]} : a;
    assign s2_r = (b[1]) ? {{2{a[31]}}, s1_r[31:2]} : s1_r;
    assign s3_r = (b[2]) ? {{4{a[31]}}, s2_r[31:4]} : s2_r;
    
    // 構建 stage1 -> stage2 中間暫存器
    reg [31:0]a_stg1, b_stg1;
    reg [2:0]op_code_stg1;
    reg [32:0]add_sub_stg1;
    reg [31:0]b_op_stg1;
    reg [31:0]shift_stg_l;
    reg [31:0]shift_stg_r;
    reg [31:0]res_stg1;
    
// 前半段資料 -> stage 1
always@(posedge clk)begin
    if(~rst_n)begin
        a_stg1 <= 0;
        b_stg1 <= 0;
        b_op_stg1 <= 0;
        op_code_stg1 <= 0;
        add_sub_stg1 <= 0;
        shift_stg_l <= 0;
        shift_stg_r <= 0;
        res_stg1 <= 0;
    end
    else begin
        a_stg1 <= a;
        b_stg1 <= b;
        b_op_stg1 <= b_op;
        op_code_stg1 <= op_code;
        
        // Stage 1 行為
        case(op_code)
            3'b000 : add_sub_stg1 <= add_sub; // +、- 法運算結果存儲
            3'b001 : add_sub_stg1 <= add_sub; // +、- 法運算結果存儲
            3'b010 : res_stg1 <= a & b;
            3'b011 : res_stg1 <= a | b;
            3'b100 : res_stg1 <= a ^ b;
            3'b101 : shift_stg_l <= s3_l; // b[0] ~ b[3] 移位結果存儲
            3'b110 : shift_stg_r <= s3_r; // b[0] ~ b[3] 移位結果存儲
            3'b111 : res_stg1 <= ($signed(a) < $signed(b)) ? 32'd1 : 32'd0;
            default : res_stg1 <= 0;
        endcase      
    end   
end
// --- stag1：前半段運算　＋　暫存器（stag1 registers）結束---

// --- stag2：後半段運算　＋　暫存器（stag2 registers）---
    
    //構建2個 32bit 暫存器用於 barrel shifter
    //Shamt 選用 b_stg1[3] ~ b_stg1[4] 判斷移動幾個 bit
    wire [31:0]s4_l, shift_l_res;
    wire [31:0]s4_r, shift_r_res;
    
    // b_stg1[3] ~ b_stg1[4] 的 logic_shift_left
    assign s4_l = (b_stg1[3]) ? {shift_stg_l[23:0], 8'b00000000} : shift_stg_l;
    assign shift_l_res = (b_stg1[4]) ? {s4_l[15:0], 16'b0000000000000000} : s4_l;
    // b_stg1[3] ~ b_stg1[4] 的 arithmetic_shift_right
    assign s4_r = (b_stg1[3]) ? {{8{a_stg1[31]}}, shift_stg_r[31:8]} : shift_stg_r;
    assign shift_r_res = (b_stg1[4]) ? {{16{a_stg1[31]}}, s4_r[31:16]} : s4_r;
    
    // 構建進位與溢位判斷
    wire carry_stg1;
    wire overflow_stg1;
    assign carry_stg1 = add_sub_stg1[32]; //進位判斷 
    assign overflow_stg1 = (~(a_stg1[31] ^ b_op_stg1[31])) && (a_stg1[31] ^ add_sub_stg1[31]); //溢位判斷
    
    // stage 1 -> stage 2（final_res）
    reg [31:0] final_res;
    always @(*) begin
        case(op_code_stg1)
            3'b000 : final_res = add_sub_stg1[31:0];
            3'b001 : final_res = add_sub_stg1[31:0];
            3'b010 : final_res = res_stg1;
            3'b011 : final_res = res_stg1;
            3'b100 : final_res = res_stg1;
            3'b101: final_res = shift_l_res;
            3'b110: final_res = shift_r_res;
            3'b111 : final_res = res_stg1;
            default: final_res = 0;
        endcase
    end
    
// stage 2 -> resault
always@(posedge clk)begin
    if(~rst_n)begin
        res <= 0;
        flag <= 0;
    end
    else begin
        res <= final_res; // 直接用統一選好的值，case 不用重複寫兩次
        // Z/N 對每種運算都更新，C/V 只在 ADD/SUB 才有意義
        flag <= {(final_res == 32'd0), final_res[31],
         ((op_code_stg1==3'b000)||(op_code_stg1==3'b001)) ? carry_stg1    : 1'b0,
         ((op_code_stg1==3'b000)||(op_code_stg1==3'b001)) ? overflow_stg1 : 1'b0};
    end   
end
// --- stag2：後半段運算　＋　暫存器（stag2 registers）結束---   
endmodule
```

#### 2. alu_v2_tt
```verilog
module alu_v2_tt();
  
  reg clk;
  reg rst_n;
  reg [31:0]a, b;
  reg [2:0]op_code;
  wire [31:0]res;
  wire [3:0]flag;
  integer i;
  
  integer error_count; // 用於計算"error"次數
  integer test_count; // 用於計算"test"次數
  
  alu_v2 tt(
  .clk(clk),
  .rst_n(rst_n),
  .a(a),
  .b(b),
  .op_code(op_code),
  .res(res),
  .flag(flag)
  );
  
  task automatic check(
      input [31:0]t_a, // a的測試變數
      input [31:0]t_b, // b的測試變數
      input [2:0]t_op, // op_code的測試變數
      input [31:0]t_res, // res的測試變數
      input t_z, t_n, t_c, t_v // flag的測試變數
  );
      begin
          a = t_a; b = t_b; op_code = t_op;
          
          @(posedge clk); // 第1拍：鎖進 Stage1
          @(posedge clk); // 第2拍：鎖進 Stage2
          #1; // 確保讀值時 NBA（非阻塞賦值） 已經真正落地
          
          test_count = test_count + 1; // 每次開始測試 test_count 次數 +1
          if(res !== t_res || flag !== {t_z, t_n, t_c, t_v})begin // res 結果或 flag 結果不符
              error_count = error_count + 1; // 錯誤就把 error_count 次數 +1
              $display("[FAIL]：%0d, op_code = %b, a = %h, b = %h | got res = %h, flag = %b | t_res = %h, flag = %b",
              test_count, t_op, t_a, t_b, res, flag, t_res, {t_z, t_n, t_c, t_v});
          end
          else begin
              $display("[PASS]：%0d, op_code = %b", test_count, t_op);
          end
      end
  endtask
  
  initial clk = 0;
  always #5 clk = ~clk;   // 每5ns翻轉一次，週期10ns

  initial begin
      rst_n = 0;
      repeat (2) @(posedge clk);   // 讓 reset 訊號至少過2個clock edge
      rst_n = 1;
      @(posedge clk); // 多留一拍緩衝，讓 reset 解除跟第一筆資料錯開
      
      error_count = 0;
      test_count = 0;
      
      // 基本功能：ADD / SUB 
      check(32'h0000_00F0, 32'h0000_000F, 3'b000, 32'h0000_00FF, 0,0,0,0); // ADD
      check(32'h0000_00F0, 32'h0000_000F, 3'b001, 32'h0000_00E1, 0,0,1,0); // SUB
      
      // 邏輯運算：AND / OR / XOR
      check(32'hFF00_FF00, 32'h0F0F_0F0F, 3'b010, 32'h0F00_0F00, 0,0,0,0); // AND
      check(32'hFF00_FF00, 32'h0F0F_0F0F, 3'b011, 32'hFF0F_FF0F, 0,1,0,0); // OR
      check(32'hFF00_FF00, 32'h0F0F_0F0F, 3'b100, 32'hF00F_F00F, 0,1,0,0); // XOR
      
      // 針對電路容易出錯的地方各自設計測資
      check(32'hF123_4567, 32'h0000_0004, 3'b110, 32'hFF12_3456, 0,1,0,0); // SRA 符號延伸
      check(32'h7FFF_FFFF, 32'h0000_0001, 3'b000, 32'h8000_0000, 0,1,0,1); // ADD overflow
      check(32'hFFFF_FFFF, 32'h0000_0005, 3'b111, 32'h0000_0001, 0,0,0,0); // SLT：-1 < 5
      check(32'h0000_0001, 32'h0000_0000, 3'b101, 32'h0000_0001, 0,0,0,0); // shamt=0
      check(32'h0000_0001, 32'h0000_001F, 3'b101, 32'h8000_0000, 0,1,0,0); // shamt=31
      
      $display("\n測試完成：共 %0d 組，失敗 %0d 組", test_count, error_count);
      if(error_count == 0)begin
          $display("ALL TESTS PASSED");
      end
      $finish;       
  end  
```

#### 3. alu_v2_wrapper
```verilog
module alu_v2_wrapper(
    
    input clk,
    input rst_n,
    input [31:0]a_in,
    input [31:0]b_in,
    input [2:0]op_in,
    output [31:0]res_out,
    output [3:0]flag_out
    
    );
    
    reg [31:0]a_r, b_r;
    reg [2:0]op_r;
    reg rst_r;
    
    always@(posedge clk)begin
        a_r <= a_in;
        b_r <= b_in;
        op_r <= op_in;
        rst_r <= rst_n;
    end
    
    alu_v2 core(
        .clk(clk),
        .rst_n(rst_r),
        .a(a_r),
        .b(b_r),
        .op_code(op_r),
        .res(res_out),
        .flag(flag_out)
    );
    
endmodule
```

---

### 驗證與硬體除錯實務 (Verification & Debugging)

#### Self-Checking Testbench 設計
1. 涵蓋 ADD/SUB, AND/OR/XOR, SRA 符號延伸、ADD Overflow 邊界案例、SLT 有號數比較 與 Shamt 極值 (0, 31)。
2. 使用 task automatic check 實現自動比對與錯誤計數。
* ex. ALU_V2（Pipeline） simulation sources
  ```verilog
  module alu_v2_tt();
    
    reg clk;
    reg rst_n;
    reg [31:0]a, b;
    reg [2:0]op_code;
    wire [31:0]res;
    wire [3:0]flag;
    integer i;
    
    integer error_count; // 用於計算"error"次數
    integer test_count; // 用於計算"test"次數
    
    alu_v2 tt(
    .clk(clk),
    .rst_n(rst_n),
    .a(a),
    .b(b),
    .op_code(op_code),
    .res(res),
    .flag(flag)
    );
    
    task automatic check(
        input [31:0]t_a, // a的測試變數
        input [31:0]t_b, // b的測試變數
        input [2:0]t_op, // op_code的測試變數
        input [31:0]t_res, // res的測試變數
        input t_z, t_n, t_c, t_v // flag的測試變數
    );
        begin
            a = t_a; b = t_b; op_code = t_op;
            
            @(posedge clk); // 第1拍：鎖進 Stage1
            @(posedge clk); // 第2拍：鎖進 Stage2
            #1; // 確保讀值時 NBA（非阻塞賦值） 已經真正落地
            
            test_count = test_count + 1; // 每次開始測試 test_count 次數 +1
            if(res !== t_res || flag !== {t_z, t_n, t_c, t_v})begin // res 結果或 flag 結果不符
                error_count = error_count + 1; // 錯誤就把 error_count 次數 +1
                $display("[FAIL]：%0d, op_code = %b, a = %h, b = %h | got res = %h, flag = %b | t_res = %h, flag = %b",
                test_count, t_op, t_a, t_b, res, flag, t_res, {t_z, t_n, t_c, t_v});
            end
            else begin
                $display("[PASS]：%0d, op_code = %b", test_count, t_op);
            end
        end
    endtask
    
    initial clk = 0;
    always #5 clk = ~clk;   // 每5ns翻轉一次，週期10ns

    initial begin
        rst_n = 0;
        repeat (2) @(posedge clk);   // 讓 reset 訊號至少過2個clock edge
        rst_n = 1;
        @(posedge clk); // 多留一拍緩衝，讓 reset 解除跟第一筆資料錯開
        
        error_count = 0;
        test_count = 0;
        
        // 基本功能：ADD / SUB 
        check(32'h0000_00F0, 32'h0000_000F, 3'b000, 32'h0000_00FF, 0,0,0,0); // ADD
        check(32'h0000_00F0, 32'h0000_000F, 3'b001, 32'h0000_00E1, 0,0,1,0); // SUB
        
        // 邏輯運算：AND / OR / XOR
        check(32'hFF00_FF00, 32'h0F0F_0F0F, 3'b010, 32'h0F00_0F00, 0,0,0,0); // AND
        check(32'hFF00_FF00, 32'h0F0F_0F0F, 3'b011, 32'hFF0F_FF0F, 0,1,0,0); // OR
        check(32'hFF00_FF00, 32'h0F0F_0F0F, 3'b100, 32'hF00F_F00F, 0,1,0,0); // XOR
        
        // 針對電路容易出錯的地方各自設計測資
        check(32'hF123_4567, 32'h0000_0004, 3'b110, 32'hFF12_3456, 0,1,0,0); // SRA 符號延伸
        check(32'h7FFF_FFFF, 32'h0000_0001, 3'b000, 32'h8000_0000, 0,1,0,1); // ADD overflow
        check(32'hFFFF_FFFF, 32'h0000_0005, 3'b111, 32'h0000_0001, 0,0,0,0); // SLT：-1 < 5
        check(32'h0000_0001, 32'h0000_0000, 3'b101, 32'h0000_0001, 0,0,0,0); // shamt=0
        check(32'h0000_0001, 32'h0000_001F, 3'b101, 32'h8000_0000, 0,1,0,0); // shamt=31
        
        $display("\n測試完成：共 %0d 組，失敗 %0d 組", test_count, error_count);
        if(error_count == 0)begin
            $display("ALL TESTS PASSED");
        end
        $finish;       
    end  
  ```
---

### 關鍵 Debug 經驗與原理
1. Verilog Event Queue Race Condition 解決：
  * 問題： Testbench 與 RTL 同步在 posedge clk 採樣，由於非阻塞賦值（NBA）在同點更新，導致讀取到舊值（Race condition）。
  * 解法： 在比對前加入 #1 時間延遲（@(posedge clk); #1;），確保讀值時 NBA 已真正落地（Settled）。
  	* 在 testbench 中加入 #1 delay 屬於 Verilog 模擬中常見的 race condition 規避手法，因為非阻塞賦值（NBA）在同一時脈邊緣的實際更新時機晚於觸發，此為 IEEE 1364 Verilog 標準中定義的排程語意（scheduling semantics）。
    ```verilog
    @(posedge clk); // 第1拍：鎖進 Stage1
    @(posedge clk); // 第2拍：鎖進 Stage2
    #1; // 確保讀值時 NBA（非阻塞賦值） 已經真正落地
    ```

2. 純組合邏輯 STA 靜態時序分析（Wrapper 設計）：
  * 問題： Baseline 版本缺少 Clock，無法建立 Register-to-Register Timing Path。
  * 解法： 包裹一層 alu_v1_wrapper 加入輸入/輸出暫存器，確立 Timing Path 進行精準 STA。
    ```verilog
    module alu_v1_wrapper(
    input clk,
    input [31:0]a_in,
    input [31:0]b_in,
    input [2:0]op_in,
    output reg [31:0]res_out,
    output reg  [3:0]flag_out
    );
    
    reg [31:0] a_r; // 鎖住輸入的暫存器，posedge clk 時取樣 a_in/b_in
    reg [31:0] b_r;
    reg [2:0] op_r; // 鎖住輸入的暫存器，posedge clk 時取樣 op_in
    wire [31:0] res_w; // alu_v1 算出的組合邏輯結果（尚未鎖存）
    wire [3:0] flag_w; // alu_v1 算出的組合邏輯 flag（尚未鎖存）
    
    always@(posedge clk)begin
        /*
        輸入暫存器：把當下的 a_in/b_in/op_in 取樣鎖住，讓 alu_v1 core 拿到的是穩定值
        ，而不是隨時可能變動的外部輸入
        */
        a_r <= a_in; 
        b_r <= b_in;
        op_r <= op_in;
        // 輸出暫存器：把 alu_v1 算好的組合邏輯結果取樣鎖住，才能輸出穩定的 res_out/flag_out
        res_out <= res_w;
        flag_out <= flag_w;
    end
    
    // alu_v1：純組合邏輯核心，接的是已經被鎖住的 a_r/b_r/op_r（不是原始輸入a_in、b_in、op_in）
    alu_v1 core(
    .a(a_r),
    .b(b_r),
    .op_code(op_r),
    .res(res_w),
    .flag(flag_w) 
    );

    endmodule
    ```

3. alu_v2_wrapper
* 問題： 初測的 alu_v2（8/5版本） 沒有輸入暫存器，輸入埠直接進入 Stage 1 邏輯（加減法器與移位器前 3 階），這段在 STA 中沒有起點暫存器，不會被量到，只量到 Stage 1 暫存器 → Stage 2 暫存器的路徑。
* 解法： 比照 alu_v1_wrapper，在 alu_v2 外加一層輸入暫存器，讓 Stage 1 的輸入也來自暫存器，Stage 1 與 Stage 2 的核心邏輯都落在暫存器到暫存器路徑內。
```verilog
module alu_v2_wrapper(
    
    input clk,
    input rst_n,
    input [31:0]a_in,
    input [31:0]b_in,
    input [2:0]op_in,
    output [31:0]res_out,
    output [3:0]flag_out
    
    );
    
    reg [31:0]a_r, b_r;
    reg [2:0]op_r;
    reg rst_r;
    
    always@(posedge clk)begin
        a_r <= a_in;
        b_r <= b_in;
        op_r <= op_in;
        rst_r <= rst_n;
    end
    
    alu_v2 core(
        .clk(clk),
        .rst_n(rst_r),
        .a(a_r),
        .b(b_r),
        .op_code(op_r),
        .res(res_out),
        .flag(flag_out)
    );
    
endmodule
```
* 說明：
	- V1、V2 的量測方式一致：輸入端各有暫存器，核心邏輯都被 STA 量到。
    - V2 的 res／flag 本來就是 Stage 2 的輸出暫存器，wrapper 不再另外加輸出暫存器。
    - FF 組成：V1 = 輸入 67 + 輸出 36 = 103；V2 = 輸入 68（含 rst）+ Stage 1 → Stage 2 級間 136 + 輸出 36 = 240。兩版 FF 的差（+137）幾乎全是 pipeline 的級間暫存器。
    - Latency（1 → 2 cycle）指核心的運算週期數，不含 wrapper 的輸入暫存器。
    - 目前 XDC 仍只有 create_clock，輸入埠到輸入暫存器、輸出暫存器到輸出埠的 I/O 時序尚未約束，列在未來優化。
	
---
### 32bit_ALU_V1（Baseline） - Synthesis + Implementation 結果，並記錄收斂到 WNS 接近 0 的結果

#### ALU_V1（Baseline）最終基準線數據
<img width="214" height="251" alt="image" src="https://github.com/user-attachments/assets/806079cd-4cc0-4a13-b4b7-5297b3a1180a" />
<img width="2229" height="198" alt="image" src="https://github.com/user-attachments/assets/37cc12a9-a720-4066-b4b0-1da357f745d1" />
<img width="1803" height="487" alt="image" src="https://github.com/user-attachments/assets/94f08a07-f7a5-4252-8ef5-b7cab7067b97" />

* Period = 6.3ns, WNS = 0.064ns
* Data path delay（critical path）：6.245ns
* Fmax ≈ 160.35 MHz（以 Period − WNS 反推）
* LUT = 349, FF = 103

### 32bit_ALU_V2（pipeline） - Synthesis + Implementation 結果，並記錄收斂到 WNS 接近 0 的結果
#### ALU_V2（pipeline）最終基準線數據
<img width="213" height="257" alt="image" src="https://github.com/user-attachments/assets/e6e79d99-8720-451a-ae4d-08f9bbb5e3b7" />
<img width="2231" height="221" alt="image" src="https://github.com/user-attachments/assets/6bcf465b-98e7-4994-bacf-ebcfbfce3bf8" />
<img width="1784" height="526" alt="image" src="https://github.com/user-attachments/assets/9d56b235-fe75-4243-9a42-56923794aeb6" />

* Period = 3.65 ns, WNS = 0.075 ns
* Data path delay（critical path）：3.515ns
* Fmax ≈ 279.72 MHz（以 Period − WNS 反推）
* LUT = 323, FF = 240

## 未來優化方向

#### 根據最終 Timing Report 觀察：
<img width="1784" height="526" alt="image" src="https://github.com/user-attachments/assets/7d8b31c1-5eca-4765-b1be-663a7ba0677e" />

1. **高扇出控制訊號**  
   Worst Path 的 Net Delay 明顯大於 Logic Delay，且 `op_code_stg1` 等高扇出訊號達到 24~72。未來可嘗試提早解碼為 one-hot，或透過 XDC 下 `MAX_FANOUT` 約束讓工具進行 register replication。

2. **完整 Timing Constraint**  
   目前 XDC 主要只有 `create_clock`，尚未加入 `set_input_delay` / `set_output_delay`。下一步將補齊完整 interface timing，確認 3.65 ns 結果是否仍能代表完整外部路徑。

3. **硬體共用策略驗證**  
   曾嘗試（以 alu_v2 08/11 版本）讓 SLT 透過「符號位 XOR 溢位旗標」共用加減法器路徑，理論上應節省面積，但實測 LUT 反而增加（317 → 392），Fmax 也略降。此經驗再次驗證：架構層級優化必須透過實際 Synthesis 與 Timing Report 數據確認，而非僅憑理論推導。

---

## 支援指令與介面

| Opcode | 指令 | 操作          |
|--------|------|---------------|
| 000    | ADD  | A + B         |
| 001    | SUB  | A - B         |
| 010    | AND  | A & B         |
| 011    | OR   | A \| B        |
| 100    | XOR  | A ^ B         |
| 101    | SLL  | A << shamt    |
| 110    | SRA  | A >>> shamt   |
| 111    | SLT  | A < B (signed)|

- 輸入：A[31:0]、B[31:0]、Opcode[2:0]
- 輸出：Result[31:0]、Flags[3:0]（Z / N / C / V）
- 移位量取自 B[4:0]
