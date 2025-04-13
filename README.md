# may-tinh-stm32
#include "liquidcrystal_i2c.h"
#include <stdio.h>

typedef enum
{
  NUMBER_0 = 	0U,
  NUMBER_1      ,
  NUMBER_2      ,
  NUMBER_3      ,
  NUMBER_4      ,
	NUMBER_5      , 
	NUMBER_6      ,
	NUMBER_7      ,
	NUMBER_8      ,
	NUMBER_9      ,
	AC            ,
	D           ,
	C           ,
	A           ,
	B           ,
	EQ            ,
	NOT_PRESS
} KeyPadType;

typedef enum
{
  N1 = 	  0U,
  OP        ,
  N2        ,
	RES
} StateCalType;

typedef enum
{
    FALSE = 0,
    TRUE = 1
} Boolean;

void GPIO_Config(void);             // cau hinh GPIO
void I2C_LCD_Configuration(void);   // cau hinh giao tiep I2C cho màn hình LCD 
KeyPadType ScanKeypad(void);        // quét bàn phím de doc duoc phim nhan 
static KeyPadType KeyPad = NOT_PRESS;
void CheckACButton(KeyPadType AcKeyPad); // kiem tra va xu ly phim xoa AC 
void ShowInforScreen(void);               //Hien thi thong tin khoi dong 
void CheckNumberButton(KeyPadType NbKeyPad); // kiem tra va xu ly khi phím so duoc nhan 
void CheckOp(KeyPadType OpKeyPad);           // kiem tra va xu ly phep toan
void CheckRes(KeyPadType ResKeyPad);         // kiem tra và hien thi ket qua khi nhan =

static KeyPadType aKeyPad[4][4] =          // ma tran phim 
{
    {NUMBER_1,  NUMBER_4,  NUMBER_7, AC},
    {NUMBER_2, 	NUMBER_5,  NUMBER_8, NUMBER_0},
    {NUMBER_3,  NUMBER_6,  NUMBER_9, EQ},
    {A,       B,       C,      D}
};
static uint16_t aCol[4] = {GPIO_Pin_4, GPIO_Pin_5, GPIO_Pin_6, GPIO_Pin_7};   // mang chua chan cac chan GPIO tuong ung voi cac cot tren ban phim 

static float u32Number1 = 0.0;    // so thu nhat, kieu so thuc ( float)
static uint32_t u32Number2 = 0;   // so thu hai, kieu so nguyen (unint32_t)
static float fResult = 0.0;       // ket qua tinh toan kieu so thuc 
static StateCalType StateCal = N1; // Bien trang thai tinh toan 
static KeyPadType KeyPadCal = NOT_PRESS; // bien luu giá tri phím phép toan 
static Boolean ReadyResuft = FALSE;
static Boolean bCheckOp = FALSE;
static Boolean bInvalid = FALSE;
static Boolean InvalRes = TRUE;   // các bien kien tra trang thai

int main(void)
{
    GPIO_Config();
    I2C_LCD_Configuration();
    HD44780_Init(2);
	
		ShowInforScreen();       // hien thi thong tin ban dau 
	
    while (1)                 // lap lai ma ben trong vong lap mai mai  
    {
				KeyPad = ScanKeypad();  // lien tuc quet ban phim 
			  
			  CheckACButton(KeyPad);   // phim xoa
				CheckNumberButton(KeyPad);// phim so 
			  CheckOp(KeyPad);          // phep toan 
			  CheckRes(KeyPad);          // hien thi ket qua
    }
}
void CheckNumberButton(KeyPadType NbKeyPad)
{
	char Str[16]; // mang hien thi ket qua 
	
	if ((NbKeyPad == NUMBER_0) || (NbKeyPad == NUMBER_1) || (NbKeyPad == NUMBER_2) || (NbKeyPad == NUMBER_3) || \   
					(NbKeyPad == NUMBER_4) || (NbKeyPad == NUMBER_5) || (NbKeyPad == NUMBER_6) || (NbKeyPad == NUMBER_7) || \
 				 (NbKeyPad == NUMBER_8) || (NbKeyPad == NUMBER_9))    // neu phim nhan la so 
		{
			if (StateCal == N1) // nhap so thu nhat 
			{
				u32Number1 = u32Number1*10 + (float)KeyPad;      // them so moi vao u32 number 
				HD44780_SetCursor(0,0);  // dat con tro hien thi len dong 0, cot 0
				sprintf(&Str[0], "%0.0f", (double)u32Number1); // dinh dang u32 number1 thanh chuoi
				HD44780_PrintStr(&Str[0]); // hien thi chuoi len man hinh 
				KeyPad = NOT_PRESS;  // dat lai trang thai bàn phím
				bCheckOp = TRUE;     // cho phep chon phep tinh tiep theo 
			}
			if (StateCal == OP)  // nhap so thu hai
			{
				u32Number2 = u32Number2*10 + KeyPad;// them so moi vao u32 number2 
				sprintf(&Str[0], "%d", KeyPad);     // dinh dang gia tri phim nhan thanh chuoi 
				HD44780_PrintStr(&Str[0]);          // hien thi chuoi len man hinh 
				KeyPad = NOT_PRESS;                 // dat lai trang thai ban phim 
				ReadyResuft = TRUE;                 // danh dau dã san sang de tinh ket qua 
				InvalRes = FALSE;                   // dat trang thai dau vao hop le 
			}
		}
}
void CheckRes(KeyPadType ResKeyPad)   // tinh toan ket qua
{
	char Str[16];
	
	if (ResKeyPad == EQ) // neu phim nhan la phim =
  {
  	if (KeyPadCal == D) // phep chia
		{
			fResult = (float)(u32Number1)/(float)u32Number2;
		}
		else if (KeyPadCal == C) // phep nhan
		{
			fResult = u32Number1 * (float)u32Number2;
		}
		else if (KeyPadCal == A) // phep cong
		{
			fResult = u32Number1 + (float)u32Number2;
		}
		else if (KeyPadCal == B)  // phep tru 
		{
			fResult = u32Number1 - (float)u32Number2;
		
		}
		else
		{
			/* Do nothing */
		}
		if (((KeyPadCal == D) && (u32Number2 == 0)) || (InvalRes == TRUE))
		{
			HD44780_SetCursor(0,1);   // dat vi tri con tro 
			HD44780_PrintStr("error");  // in ra man hinh neu gia tri khong hop le 
			bInvalid = TRUE;
		}
		else
		{
			if ((KeyPadCal == D) && (u32Number1 < (float)u32Number2))
			{
				HD44780_SetCursor(0,1);
				if ((u32Number1 != (uint32_t)u32Number1))
				{
					sprintf(&Str[0], "%0.1f", (double)fResult);
				}
				else
				{
					sprintf(&Str[0], "%0.0f", (double)fResult);
				}
				HD44780_PrintStr(&Str[0]);
			}
			else
			{
				if ((KeyPadCal == D) || (u32Number1 != (uint32_t)u32Number1))
				{
					HD44780_SetCursor(0,1);
					sprintf(&Str[0], "%0.1f", (double)fResult);
					HD44780_PrintStr(&Str[0]);
				}
				else
				{
					HD44780_SetCursor(0,1);
					sprintf(&Str[0], "%0.0f", (double)fResult);
					HD44780_PrintStr(&Str[0]);
				}
			}
		}
		StateCal = RES;
		ReadyResuft = FALSE;
  }
}
void CheckOp(KeyPadType OpKeyPad)
{
	char Str[16];
	
	if ((bInvalid == FALSE) && (ReadyResuft != TRUE))  // chi xu ly khi khong co loi va chua san sang ket qua  
	{
		if ((OpKeyPad == B) || ((OpKeyPad == B) && (StateCal == RES)))  // phep tru
		{
			if (StateCal == RES)  // neu trang thai la ket qua 
			{
				ShowInforScreen();  // xoa man hinh 
				u32Number1 = fResult; // lay ket qua lam so thu nhat 
				u32Number2 = 0U;      // dat so thu hai ve 0
				HD44780_SetCursor(0,0);  // hien thi so thu nhat 
				if (u32Number1 == (uint32_t)u32Number1)
				{
					sprintf(&Str[0], "%0.0f", (double)u32Number1);
				}
				else
				{
					sprintf(&Str[0], "%0.1f", (double)u32Number1);
				}
				HD44780_PrintStr(&Str[0]);
			}
			if (bCheckOp == TRUE)
			{
				ShowInforScreen();
				HD44780_SetCursor(0,0);
				if (u32Number1 == (uint32_t)u32Number1)
				{
					sprintf(&Str[0], "%0.0f", (double)u32Number1);
				}
				else
				{
					sprintf(&Str[0], "%0.1f", (double)u32Number1);
				}
				HD44780_PrintStr(&Str[0]);
				HD44780_PrintStr("-");
				StateCal = OP;
				KeyPadCal = B;
			}
			else
			{
				HD44780_SetCursor(0,1);
				HD44780_PrintStr("error");
				bInvalid = TRUE;
			}
		}
		else if ((OpKeyPad == C) || ((OpKeyPad == C) && (StateCal == RES)))
		{
			if (StateCal == RES)
			{
				ShowInforScreen();
				u32Number1 = fResult;
				u32Number2 = 0U;
				HD44780_SetCursor(0,0);
				if (u32Number1 == (uint32_t)u32Number1)
				{
					sprintf(&Str[0], "%0.0f", (double)u32Number1);
				}
				else
				{
					sprintf(&Str[0], "%0.1f", (double)u32Number1);
				}
				HD44780_PrintStr(&Str[0]);
			}
			if (bCheckOp == TRUE)
			{
				ShowInforScreen();
				HD44780_SetCursor(0,0);
				if (u32Number1 == (uint32_t)u32Number1)
				{
					sprintf(&Str[0], "%0.0f", (double)u32Number1);
				}
				else
				{
					sprintf(&Str[0], "%0.1f", (double)u32Number1);
				}
				HD44780_PrintStr(&Str[0]);
				HD44780_PrintStr("*");
			StateCal = OP;
			KeyPadCal = C;
			}
			else
			{
				HD44780_SetCursor(0,1);
				HD44780_PrintStr("error");
				bInvalid = TRUE;
			}
			
		}
		else if ((OpKeyPad == A) || ((OpKeyPad == A) && (StateCal == RES)))
		{
			if (StateCal == RES)
			{
				ShowInforScreen();
				u32Number1 = fResult;
				u32Number2 = 0U;
				HD44780_SetCursor(0,0);
				if (u32Number1 == (uint32_t)u32Number1)
				{
					sprintf(&Str[0], "%0.0f", (double)u32Number1);
				}
				else
				{
					sprintf(&Str[0], "%0.1f", (double)u32Number1);
				}
				HD44780_PrintStr(&Str[0]);
			}
			if (bCheckOp == TRUE)
			{
				ShowInforScreen();
				HD44780_SetCursor(0,0);
				if (u32Number1 == (uint32_t)u32Number1)
				{
					sprintf(&Str[0], "%0.0f", (double)u32Number1);
				}
				else
				{
					sprintf(&Str[0], "%0.1f", (double)u32Number1);
				}
				HD44780_PrintStr(&Str[0]);
				HD44780_PrintStr("+");
				StateCal = OP;
				KeyPadCal = A;
			}
			else
			{
				HD44780_SetCursor(0,1);
				HD44780_PrintStr("error");
				bInvalid = TRUE;
			}
			
		}
		else if ((OpKeyPad == D) || ((OpKeyPad == D) && (StateCal == RES)))
		{
			if (StateCal == RES)
			{
				ShowInforScreen();
				u32Number1 = fResult;
				u32Number2 = 0U;
				HD44780_SetCursor(0,0);
				if (u32Number1 == (uint32_t)u32Number1)
				{
					sprintf(&Str[0], "%0.0f", (double)u32Number1);
				}
				else
				{
					sprintf(&Str[0], "%0.1f", (double)u32Number1);
				}
				HD44780_PrintStr(&Str[0]);
			}
			if (bCheckOp == TRUE)
			{
				ShowInforScreen();
				HD44780_SetCursor(0,0);
				if (u32Number1 == (uint32_t)u32Number1)
				{
					sprintf(&Str[0], "%0.0f", (double)u32Number1);
				}
				else
				{
					sprintf(&Str[0], "%0.1f", (double)u32Number1);
				}
				HD44780_PrintStr(&Str[0]);
				HD44780_PrintStr(":");
				StateCal = OP;
				KeyPadCal = D;
			}
			else
			{
				HD44780_SetCursor(0,1);
				HD44780_PrintStr("error");
				bInvalid = TRUE;
			}
		}
		else 
		{
			/* Do Thing */
		}
	}
}
void CheckACButton(KeyPadType AcKeyPad)  
{
	if (AcKeyPad == AC)  // neu phim AC duoc nhan 
  {
  	u32Number1 = 0;    // dat so thu nhat ve 0
  	u32Number2 = 0;    // dat so thu hai ve 0
		StateCal = N1;     // chuyen trang thai ve nhap so thu nhat
		ReadyResuft = FALSE; // dat trang thai chua san sang tinh ket qua
		bCheckOp = FALSE;    // dat trang thai chua chon phep tinh 
		bInvalid = FALSE;    // dat trang thai khong co loi
		InvalRes = TRUE;     // dat trang thai dau vao hop le
		ShowInforScreen();   // xoa man hinh va hien thi thong tin ban dau 
  }
}
void ShowInforScreen(void)
{
	HD44780_Clear();
	HD44780_SetCursor(0,0);
	HD44780_PrintStr(" ");
	HD44780_SetCursor(0,1);
	HD44780_PrintStr(" ");
	HD44780_SetCursor(0,0);
	HD44780_Blink();
}
KeyPadType ScanKeypad(void)
{
	uint8_t Count = 0;
  KeyPadType TempKeyPad = NOT_PRESS;
	
	for (Count = 0; Count < 4; Count++)
	{
		GPIO_ResetBits(GPIOA, aCol[Count]);
		if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0) == 0U)
		{
			Delay_Ms(200);
			if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0) == 0U)
			{
					TempKeyPad = aKeyPad[Count][0];
			}
		}
		else if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_1) == 0U)
		{
			Delay_Ms(200);
			if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_1) == 0U)
			{
					TempKeyPad = aKeyPad[Count][1];
			}
		}
		else if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_2) == 0U)
		{
			Delay_Ms(200);
			if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_2) == 0U)
			{
					TempKeyPad = aKeyPad[Count][2];
			}
		}
		else if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_3) == 0U)
		{
			Delay_Ms(200);
			if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_3) == 0U)
			{
					TempKeyPad = aKeyPad[Count][3];
			}
		}
		else
		{
			/* Do nothing */
		}
		
		GPIO_SetBits(GPIOA, aCol[Count]);
	}
	
	return TempKeyPad;
}
void GPIO_Config(void)
{
    GPIO_InitTypeDef  GPIO_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0|GPIO_Pin_1|GPIO_Pin_2|GPIO_Pin_3;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_4|GPIO_Pin_5|GPIO_Pin_6|GPIO_Pin_7;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
		
		GPIO_SetBits(GPIOA, GPIO_Pin_4|GPIO_Pin_5|GPIO_Pin_6|GPIO_Pin_7);
}
void I2C_LCD_Configuration(void)
{
    GPIO_InitTypeDef  GPIO_InitStructure;
    I2C_InitTypeDef  I2C_InitStructure;

    /* PortB clock enable */
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
    /* I2C1 clock enable */
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C1, ENABLE);  
    GPIO_StructInit(&GPIO_InitStructure);            

    /* SCL (PB6) */
    /* SDA (PB7) */
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6 | GPIO_Pin_7;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;      
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_OD;  
    GPIO_Init(GPIOB, &GPIO_InitStructure);           

    /* Configure I2Cx */
    I2C_StructInit(&I2C_InitStructure);  
    I2C_InitStructure.I2C_Mode = I2C_Mode_I2C;
    I2C_InitStructure.I2C_DutyCycle = I2C_DutyCycle_2; 
    I2C_InitStructure.I2C_OwnAddress1 = 0x00;
    I2C_InitStructure.I2C_Ack = I2C_Ack_Enable;
    I2C_InitStructure.I2C_AcknowledgedAddress = I2C_AcknowledgedAddress_7bit;
    I2C_InitStructure.I2C_ClockSpeed = 100000;

    I2C_Init(I2C_Chanel, &I2C_InitStructure);
    I2C_Cmd(I2C_Chanel, ENABLE);
}
