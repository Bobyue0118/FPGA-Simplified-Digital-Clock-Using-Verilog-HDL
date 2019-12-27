# Simplified-Digital-Clock-Using-Verilog-HDL
## Info
- This is an experiment course, called "Electronic Technology".  
- This is the big project of FPGA.  
## Dependencies
- ISE
- PlanAhead
- Adept

**********************************************************
# 简易数字钟的制作和调试实验报告 #

- **姓名**：岳博
- **学号**：517021910825
- **邮箱**：465741287@qq.com
- **上课时间**：周一下午1:00-4:00



## 功能介绍 ##

### 基本功能 ###

#### 时间显示 ####

- 时间显示以秒为单位计时，默认以24小时制显示。
- 右7位LED灯（即LD6 ~ LD0）以BCD码的方式表示当前秒数，其中（LD6 ~ LD4）表示秒的十位部分，（LD3 ~ LD0）表示秒的个位部分。
- 小时数和分钟数则分别显示在数码管的左两位和右两位。每隔一秒位于中间的小数点完成一次闪烁。

#### 时间设置 ####

- 按下*BTN3*进入时间设置模式。首先设置的是秒，设置完后再按下*BTN3*可以设置分钟，之后再次按下*BTN3*设置时钟，这时再按一次便回到普通的时间显示模式，显示的时间即为更新后的值。

- 设置时使用开关*SW[7:0]*，高四位对应十位数，低四位对应个位数（均以BCD码的形式），实时数据显示在对应LED或数码管上。当输入非法值时（*比如设置秒数时输入了61，或者设置小时数为25* ），FPGA可以做出判断自动把对应位数强制置零，置零结果也同步显示在相应的LED（秒）或者数码管（小时或者分钟）上。

#### 整点报时 ####

- 当时间来到整点时，最左边的LED灯（LD7）开始闪烁，作整点报时。



### 扩展功能 ###

#### 时制切换 ####

- 按下*BTN2*后可以在24小时制和12小时制的时间之间切换。

#### 闹铃功能 ####

- 按下*BTN1*进入闹钟设置模式。
- 设置闹钟时只设置分钟和时钟，不设置秒。设置的细节和时间设置相同。设置好后回到时间显示模式。
- 闹钟起始值默认为中午12点。到达闹钟设置时间后，数码管上显示的四位数字不变，但开始以1秒为周期闪烁。*注：考虑到没有更多的输出设备以区别上午下午，故12小时下的闹钟设置依然按24小时的模式显示。（即当12小时制和24小时制下，时钟显示不同时，闹钟设置的是24小时制下的闹钟）*

#### 清零功能 ####

- 按下*BTN0*后的当前时间变为*00:00:00*，时间显示模式变为24小时制，闹钟重新设在中午12点。



## 代码说明 ##

- 基本的数码管显示模块，和之前四个实验中的代码一致。
```verilog
module clk_7segff_sub(
 	input [3:0]NUM,
 	output reg[6:0]a_to_g
 	);
 	
 	always @(*)
 	case(NUM)
 		0:a_to_g=7'b0000001;
 		1:a_to_g=7'b1001111;
 		2:a_to_g=7'b0010010;
 		3:a_to_g=7'b0000110;
 		4:a_to_g=7'b1001100;
 		5:a_to_g=7'b0100100;
 		6:a_to_g=7'b0100000;
 		7:a_to_g=7'b0001111;
 		8:a_to_g=7'b0000000;
 		9:a_to_g=7'b0000100;
 		'hA: a_to_g=7'b0001000;
 		'hB: a_to_g=7'b1100000;
 		'hC: a_to_g=7'b0110001;
 		'hD: a_to_g=7'b1000010;
 		'hE: a_to_g=7'b0110000;
 		'hF: a_to_g=7'b0111000;
 		default: a_to_g=7'b0000001;
 	endcase
 	endmodule
```



- 主模块输入输出端口定义
```verilog
module DigitalClock_top(
		input [7:0]SW,
		input clk,
		input clr,			// Btn0 清零功能
		input alarm,		// Btn1 闹铃功能
		input twelve,		// Btn2 12-24小时切换
		input shift,		// Btn3 设置时间，考虑到12小时制需要设置上下午，用户体验不佳，故只能在24小时制下进行
		output [7:0]led,
		output [3:0]an,
		output [6:0]a_to_g,
		output dp
	);
```

- 辅助寄存器和接线定义
```verilog
    reg [25:0] counter;						// 计数器
	reg colon = 1;							// 表示在某段时间内小数点应该要被显示
	reg colonOfSecond = 1;					// 小数点在第二位显示用
	reg [3:0] second_one = 0;           	// 时钟的六个变量
	reg [2:0] second_ten = 0;           	// 
	reg [3:0] minute_one = 0;				// 
	reg [3:0] minute_ten = 0;				// 
	reg [3:0] hour_one = 0;					// 
	reg [3:0] hour_ten = 0;					// 
	reg [3:0] alarm_minone = 0;				// 闹钟的四个变量，初始化为中午12点
	reg [3:0] alarm_minten = 0;				//
	reg [3:0] alarm_hourone = 2;			//
	reg [3:0] alarm_hourten = 1; 			//
    wire [3:0] getalarm;					// 判断是否报时用(闹钟)
    wire [1:0] s;							// 用于一般情况下的数码管分频交替显示
	reg [1:0] m = 0;						// 用于更改时间时数码管的禁用
    reg [3:0] l = 0;						// 用于更改时间时led灯的禁用
	reg [3:0] NUM;							// 数码管模块传参用
	reg [1:0] shiftMode = 0;				// 进入设定时间模式标志位,先设定秒，再设定分钟，再设定小时
	reg twelveMode = 0;						// 12-24切换标志寄存器，为1时进入12小时制
	reg [1:0] alarmMode = 0;				// 进入设定闹钟模式标志位,先设定分钟，再设定小时
    wire [3:0] timerr;                      //整点报时专用，如果是0，LD7开始闪烁
	reg shine = 0;                          //整点报时时，LD7闪烁专用
```

- 输出管脚关联
```verilog
    assign s = counter[16:15];				       // 分频
    assign an[0] = s[0]|s[1]|m[0];			       // m用于区分左右两个数码管
	assign an[1] = ~s[0]|s[1]|m[0];
	assign an[2] = s[0]|~s[1]|m[1];
	assign an[3] = ~s[0]|~s[1]|m[1];
	assign dp = colonOfSecond;
    assign led[6:4] = second_ten|l;			       // l用于区分左右四个led管
	assign led[3:0] = second_one|l;
    assign led[7] = shine;                         //shine控制LD7的闪烁
	assign getalarm = alarm_minone^minute_one | alarm_hourone^hour_one | alarm_minten^minute_ten | alarm_hourten^hour_ten;	// 判断何时到达闹钟设定时间
    assign timerr = minute_ten^(4'b0000) |  minute_one^(4'b0000);  //判断整点报时
```
- 设置各种功能模式的触发条件
```verilog
    // 按下Btn1进入设置闹钟模式
	always @(posedge alarm) begin
		alarmMode = alarmMode + 1;
		if(alarmMode==3) alarmMode = alarmMode + 1; // 跳过无用的一项，因为闹钟不设置秒
	end
	// 按下Btn2进入修改时间模式
	always @(posedge twelve)
		twelveMode = twelveMode + 1;
	// 按下Btn3进入修改时间模式
	always @(posedge shift)
		shiftMode = shiftMode + 1;
```


**核心代码部分**

- 清零功能
```verilog
	always @(posedge clk) begin
		counter = counter + 1;
		//一键清零
		if(clr==1) begin
			counter = 0;
			second_one = 0;
			second_ten = 0;
			minute_one = 0;
			minute_ten = 0;
			hour_one = 0;
			hour_ten = 0;
			alarm_minone = 0;
			alarm_minten = 0;
			alarm_hourone = 2;	// 清零时闹钟设在中午12点整
			alarm_hourten = 1;
			m=0;
			l=0;
		end
```
- 正常显示时间模式
```verilog
		if(!shiftMode) begin
			m=0;	                                        // 初始化数码管与LED灯
			l=0;
			if(counter>50000000) begin//1 					// 每一到一秒钟是都要做如下判断
				counter = 0;							    // 0.内部时钟清零
				colon = colon + 1;						    // 1.小数点闪烁,2s一个周期
				second_one = second_one + 1;				// 2.秒数+1
				if(second_one==10) begin//2
					second_one = 0;
					second_ten = second_ten + 1;
					if(second_ten==6) begin//3
						second_ten = 0;
						minute_one = minute_one + 1;		// 3.一分钟过去了
						if(minute_one==10) begin//4
							minute_one = 0;
							minute_ten = minute_ten + 1;
							if(minute_ten==6) begin//5
								minute_ten = 0;
								hour_one = hour_one + 1;	// 4.一小时过去了
								if(hour_one==10) begin 				
									hour_one = 0;
									hour_ten = hour_ten + 1;
								end
								else if(hour_one==4) begin
									if(hour_ten==2) begin
									hour_one = 0;			//5.一天过去了
									hour_ten = 0;
									end
								end 
							end//5
						end//4
					end//3
				end//2
			end//1
			if(!alarmMode) begin
			// 判断什么时候显示数码管中间的小数点
			if(s==2) colonOfSecond = colon;
			else colonOfSecond = 1;
			// 分频后的数码管显示情况以及数码管闪烁情况
			if(getalarm==0) begin
				 case(colon)
			       1:m[1:0] = 2'b11;
                   0:m[1:0] = 2'b00;//数码管闪烁
			    endcase
				 if(!colon) begin	// 24 hour
					case(s) 
						3:NUM = hour_ten;
						2:NUM = hour_one;
						1:NUM = minute_ten;
						0:NUM = minute_one;
					endcase
				 end
			end
			else begin
	 	 		if(!twelveMode) begin	// 24 hour
					case(s) 
						3:NUM = hour_ten;
						2:NUM = hour_one;
						1:NUM = minute_ten;
						0:NUM = minute_one;
					endcase
				end
				else begin			// 12 hour
					case(s) 
						3: 	if(hour_ten==1) begin
								if(hour_one<3) NUM = hour_ten;
								else NUM = hour_ten - 1; 
							end
							else begin
								if(hour_ten==0) NUM = hour_ten; 
								else begin
									if(hour_ten==2) 
										if(hour_one==0) NUM = 4'b0000;
										else if(hour_ten==2) 
											if(hour_one==1) NUM = 4'b0000;
											else NUM = 1;
								end
							end
						2: 	if(hour_ten==1)
								if(hour_one<3) NUM = hour_one;
								else NUM = hour_one - 2; 
							else begin
								if(hour_ten==0) NUM = hour_one; 
								else begin
									if(hour_ten==2) 
										if(hour_one==0) NUM = 4'b1000;
										else if(hour_ten==2) 
											if(hour_one==1) NUM = 4'b1001;
											else NUM = hour_one - 2;
								end
							end
						1:	NUM = minute_ten;
						0:	NUM = minute_one;
					endcase
				end
			end
		end
		end
```

- 整点报时
```verilog
  		if(timerr==0) begin
    		case(colon)
    		1: shine = 1;
    		0: shine = 0;
    		endcase
    	end
    	if(timerr!=0) shine = 0;
```

- 时间设置模式
```verilog
	 	// 设定秒数，显示在led上
	 	if(alarmMode==0) begin
		if(shiftMode==1) begin
			m[1:0] = 2'b11;	// 数码管禁用
			l = 4'b0000;	// LED启用
			if(SW[6:4]>5) second_ten = 0; else second_ten = SW[6:4];
			if(SW[3:0]>9) second_one[3:0] = 0; else second_one[3:0] = SW[3:0];
		end
		end
	 	// 设置分钟数，显示在数码管右两位
	 	if(alarmMode==0) begin
		if(shiftMode==2) begin
			l = 4'b1111;		// LED禁用
			m[1:0] = 2'b10;		// 数码管禁用左二位
			// 设置数码管显示方案，超过一定数值自变零
			case(counter[15]) 
				1: if(SW[7:4]<6) NUM = SW[7:4]; else NUM = 0;
				0: if(SW[3:0]<10) NUM = SW[3:0]; else NUM = 0;
			endcase
           // 因为数码管显示是跳变的(NUM的值会改变），所以要设置稳定的赋值方案，使之与数码管的显示值一致
			if(SW[7:4]<6) minute_ten = SW[7:4]; else minute_ten = 0;
			if(SW[3:0]<10) minute_one = SW[3:0]; else minute_one = 0;
		end
		end
	 	// 设置小时数，小时在数码管左两位
	 	if(alarmMode==0) begin
		if(shiftMode==3) begin
			l = 4'b1111;		// LED禁用
			m[1:0] = 2'b01;		// 数码管禁用右二位
			// 设置数码管显示方案，超过一定数值自变零,与设置分钟数时稍有不同
			case(counter[15]) 
				1: if(SW[7:4]<3) NUM = SW[7:4]; else NUM = 0;
				0: if(SW[7:4]==2) 
						if(SW[3:0]<4) NUM = SW[3:0]; else NUM = 0;
					else 
						if(SW[3:0]<10) NUM = SW[3:0]; else NUM = 0;
			endcase
			// 设置赋值方案，与数码管的显示值一致,与设置分钟数时稍有不同
			if(SW[7:4]<3) hour_ten = SW[7:4]; else hour_ten = 0;
			if(SW[7:4]==2) 
				if(SW[3:0]<4) hour_one = SW[3:0]; else hour_one = 0;
			else 
				if(SW[3:0]<10) hour_one = SW[3:0]; else hour_one = 0;
		end
		end
```
- 闹钟设置模式
``` verilog
		// 设置分钟数，显示在数码管右两位
		if(shiftMode==0) begin
		if(alarmMode==1) begin
			l = 4'b1111;
			m[1:0] = 2'b10;
			case(counter[15]) 
				1: if(SW[7:4]<6) NUM = SW[7:4]; else NUM = 0;
				0: if(SW[3:0]<10) NUM = SW[3:0]; else NUM = 0;
			endcase
			if(SW[7:4]<6) alarm_minten = SW[7:4]; else alarm_minten = 0;
			if(SW[3:0]<10) alarm_minone = SW[3:0]; else alarm_minone = 0;
		end
		end
	 	// 设置小时数，小时在数码管左两位	
		if(shiftMode==0) begin
		if(alarmMode==2) begin
			l = 4'b1111;
			m[1:0] = 2'b01;
			case(counter[15]) 
				1: if(SW[7:4]<3) NUM = SW[7:4]; else NUM = 0;
				0: if(SW[7:4]==2) 
						if(SW[3:0]<4) NUM = SW[3:0]; else NUM = 0;
					else 
						if(SW[3:0]<10) NUM = SW[3:0]; else NUM = 0;
			endcase
			if(SW[7:4]<3) alarm_hourten = SW[7:4]; else alarm_hourten = 0;
			if(SW[7:4]==2) 
				if(SW[3:0]<4) alarm_hourone = SW[3:0]; else alarm_hourone = 0;
			else 
				if(SW[3:0]<10) alarm_hourone = SW[3:0]; else alarm_hourone = 0;
		end
		end
	end
	
    //调用数码管显示模块
	clk_7segff_sub A1(.NUM(NUM),.a_to_g(a_to_g));
endmodule
```

- 管脚定义
```verilog
    #PlanAhead Generated physical constraints 
    NET "alarm" CLOCK_DEDICATED_ROUTE = FALSE;
    NET "a_to_g[0]" LOC = M12;
   	NET "a_to_g[1]" LOC = L13;
   	NET "a_to_g[2]" LOC = P12;
   	NET "a_to_g[3]" LOC = N11;
   	NET "a_to_g[4]" LOC = N14;
   	NET "a_to_g[5]" LOC = H12;
   	NET "a_to_g[6]" LOC = L14;
   	NET "an[0]" LOC = F12;
   	NET "an[1]" LOC = J12;
   	NET "an[2]" LOC = M13;
   	NET "an[3]" LOC = K14;
   	NET "led[0]" LOC = M5;
   	NET "led[1]" LOC = M11;
   	NET "led[2]" LOC = P7;
   	NET "led[3]" LOC = P6;
   	NET "led[4]" LOC = N5;
   	NET "led[5]" LOC = N4;
   	NET "led[6]" LOC = P4;
   	NET "led[7]" LOC = G1;
   	NET "SW[0]" LOC = P11;
   	NET "SW[1]" LOC = L3;
   	NET "SW[2]" LOC = K3;
   	NET "SW[3]" LOC = B4;
   	NET "SW[4]" LOC = G3;
   	NET "SW[5]" LOC = F3;
   	NET "SW[6]" LOC = E2;
   	NET "SW[7]" LOC = N3;
   	NET "clk" LOC = B8;
   	NET "shift" LOC = A7;
   	NET "clr" LOC = G12;
   	NET "twelve" LOC = M4;
   	NET "alarm" LOC = C11;
   	NET "dp" LOC = N13;
```



## 实验结果 ##

- 两个基本功能以及三个扩展功能均能很好地实现。



## 实验感想 ##

- 经过三周时间的学习和debug的过程可以比较熟练地使用verilog语言了。
- debug的时间还是太长，因为每次都需要重新generate，很多时间都花费在生成.bit文件上了。如果可以有更快捷的输出响应会更好。
