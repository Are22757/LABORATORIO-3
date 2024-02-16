# LABORATORIO-3
//****************************************************
// Encabezado 
//****************************************************
.include "M328PDEF.inc"

.cseg
.org 0x00
	JMP MAIN   ; Salta al bucle principal con la configuración inicial
.org 0x0006
	JMP INT_PC  ; Salta a la rutina de interrupción de pines
.org 0x0020
	JMP INT_TIMER0  ; Salta a la rutina de interrupción de timer
//****************************************************
//Configuración de la Pila
//****************************************************
Main: 
	LDI R16, LOW(RAMEND)
	OUT SPL, R16  ; Inicializa el puntero de pila baja
	LDI R17, HIGH(RAMEND)
	OUT SPH, R17  ; Inicializa el puntero de pila alta
		
//****************************************************
//Configuración MCU
//****************************************************
SETUP:
	LDI R16, 0b1000_0000  ; Configura el reloj a 8MHz
	LDI R16, (1 << CLKPCE)
	STS CLKPR, R16

	LDI R16, 0b0000_0001
	STS CLKPR, R16
		
	LDI R16, 0b0000_0011  ; Configura el puerto B como entradas y salidas
	OUT	PORTB, R16

	LDI R16, 0b0000_1100  ; Configura el puerto B con resistencias pull-up
	OUT DDRB, R16
	
	LDI R16, 0b1011_1111  ; Configura el puerto C como salidas
	OUT DDRC, R16

	LDI R16, 0b1111_1111  ; Configura el puerto D como salidas
	OUT DDRD, R16

	LDI R16, (1<<PCINT1) | (1<<PCINT0)  ; Establece los pines 0 y 1 del puerto B como interrupciones
	STS PCMSK0, R16

	LDI R16, (1<<PCIE0)
	STS PCICR, R16

	SEI  ; Habilita las interrupciones globales

	LDI R20, 0x00
	LDI R21, 0x00  ; Inicializa registros para su uso dentro del código
	LDI R22, 0x00
	LDI R23, 0x00

	; Llama a la inicialización del timer0
	CALL INT_T0
	SBI PINB, 3  ; Enciende el PB3 para la multiplexación 
	
//****************************************************
// Bucle principal
//****************************************************

LOOP:
	; Comprueba si las unidades alcanzan 10 para aumentar las decenas
	CPI R22, 10
	BREQ DECE      

	; Comprueba si los segundos llegan a 60 para reiniciar los contadores
	CPI R23, 60
	BREQ RESET      

	; Realiza la multiplexación entre los displays
	CALL DELAY
	SBI PINB, 2    
	SBI PINB, 3

	; Muestra las decenas en el primer display
	LDI ZH, HIGH(TABLA7SEG << 1)
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, R24 
	LPM R25, Z
	OUT PORTD, R25    

	CALL DELAY
	
	SBI PINB, 2
	SBI PINB, 3   

	; Muestra las unidades en el segundo display
	LDI ZH, HIGH(TABLA7SEG << 1)
	LDI ZL, LOW(TABLA7SEG << 1)   
	ADD ZL, R22
	LPM R25, Z
	OUT PORTD, R25       

	CALL DELAY

	OUT PORTC, R20    ; Visualiza el contador en los leds a manos de los botones 
	RJMP LOOP   

DELAY:              ; Delay para la multiplexación de los displays 
	LDI R19, 255
DELAY1:
	DEC R19
	BRNE DELAY1 
	LDI R19, 255
DELAY2:
	DEC R19
	BRNE DELAY2
	LDI R19, 255
DELAY3:
	DEC R19
	BRNE DELAY3
	LDI R19, 255
	DELAY4:
	DEC R19
	BRNE DELAY4

	RET

DECE:            ; Incrementa las decenas cuando las unidades llegan a 10
	LDI R22, 0
	INC R24 
	JMP LOOP

RESET:
	CALL DELAY   ; Reinicia los contadores cuando los segundos llegan a 60
	LDI R22, 0
	LDI R23, 0
	JMP LOOP

//****************************************************
//Subrutina de interrupción para manejar los botones
//****************************************************
INT_PC:
	PUSH R16
	IN R16, SREG
	PUSH R16

	IN R18, PINB

	SBRC R18, PB0
	JMP BTTON1

	INC R20
	CPI R20, 16
	BRNE SALIR
	LDI R20, 0
	JMP SALIR

BTTON1:
	SBRC R18, PB1
	JMP SALIR

	DEC R20
	BRNE SALIR
	LDI R20, 15

SALIR:
	SBI PINB, PB5
	SBI PCIFR, PCIF0

	POP R16
	OUT SREG, R16
	POP R16
	RETI

//****************************************************
//Subrutina de inicialización del timer0
//****************************************************
INT_T0:
	LDI R26, 0
	OUT TCCR0A, R26      ; Inicializa timer 0 como contador 
	
	LDI R26, (1<<CS02) | (1<<CS00)     ; Selección de prescaler de 1024 
	OUT TCCR0B, R26       
	
	LDI R26, 100           ; Valor de conteo inicial 
	OUT TCNT0, R26

	LDI R26, (1<<TOIE0)   
	STS TIMSK0, R26

	RET

//****************************************************
//Subrutina de interrupción del timer0
//****************************************************
INT_TIMER0:
	PUSH R16        ; Guarda el valor de R16
 	IN R16, SREG
	PUSH R16

	LDI R16, 100
	OUT TCNT0, R16      
	SBI TIFR0, TOV0
	
	INC R23           ; Incrementa los segundos en cada interrupción del timer0

	POP R16
OUT SREG, R16  
	POP R16         ; Recupera el valor de R16

	RETI

//****************************************************
// Tabla de segmentos para mostrar en los displays
//****************************************************
TABLA7SEG: .DB 0x7E, 0x0C, 0xB6, 0x9E, 0xCC, 0xDA, 0xFA, 0x0E, 0xFE, 0xDE
//****************************************************
