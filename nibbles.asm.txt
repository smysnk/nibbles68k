;=============================================================================
;        File: nibbles.asm
;      Author: Joshua Bellamy-Henn
;             © 2015 Psidox
;
; Description: Remake of Nibbles game for the Motorola 68000.
;
; How to Play: Use W,S,A,D keys or (keypad 8,4,5,6) to change direction, objective 
;              is to grow the snake as long as possible by eating food placed in 
;              random locations around the arena.  Don't run into your own trail 
;              or into wall you will die!!!
;
;=============================================================================

CHAR_CR EQU     $0D

DUART   EQU     $600000
MR1A    EQU     1
SRA     EQU     3
RBA     EQU     7       ; Receiver buffer register (data in)
TBA     EQU     7       ; Transmitter buffer register (data out)
RxRDY   EQU     0       ; Reciver ready bit
TxRDY   EQU     2       ; Transmitter ready bit

DIR_LEFT  EQU   0
DIR_RIGHT EQU   1
DIR_UP    EQU   2
DIR_DOWN  EQU   3

WALL_LIFE EQU   $FFFE     ; Wall Special Time
FOOD_LIFE EQU   $FFFF     ; Food Special Time
FOOD_GROWTH EQU 15      ; How long snake will grow after eating food

SNK_START_LIFE  EQU   5      ; Life of single segment of the snake
SNK_SPEED_EASY  EQU   $0000E000
SNK_SPEED_MEDIUM EQU  $0000AFFF
SNK_SPEED_HARD EQU    $00007FFF
SNK_SPEED_INSANE EQU  $00004000

SNK_SCR_SIZE EQU 1716   ; x*y = 78*22

ARENA_X EQU     78
ARENA_Y EQU     23

START_X EQU     38
START_Y EQU     10

        ORG     $4000              
        
NIBBLES ;initialize values that retain value between games        
        MOVE.B  #1,DIFFICULTY            

intro   ;initialize values that will reset at start of each game
        MOVE.B #0,LEVEL   ; player starts at level 0 (aka level 1) initially 
        MOVE.B #0,SCORE    ; Reset Score
        MOVE.B #0,FOOD_NUM ; Reset Score
        MOVE.B #5,LIVES    ; Set starting lives 

        MOVEA.L  #STR_SPLASH_SCR,A1 ; draw splash screen
        BSR _DISPSTR
        BSR _SELECT_DIFF ; Let user select difficulty     

        
loadlevel    
        ;initialize values that will reset at start of each level / life
        MOVE.W #SNK_START_LIFE, SNK_LIFE
        MOVE.B #START_X,POS_X  ; Set starting X
        MOVE.B #START_Y,POS_Y  ; Set starting Y
        MOVE.L #0,TIMER        ; Init Timer to 0
        MOVE.W #0,RAND_MEM     ; Random Memory Location
        MOVE.B #4,DIRECTION    ; No direction
        MOVE.B #4,LAST_DIR     ; No last direction
        MOVE.B #0,MOVING
        MOVE.L #0,TIMER
        MOVE.B #0,FOOD_AVAIL
        MOVE.B #0,DELAY_DECAY

        MOVEA.L  #STR_ARENA_SCR,A1 ; draw arena
        BSR _DISPSTR
        BSR     _CLEARMEM    ; Clear Arena Memory

        CLR.L  D1
        MOVE.B LEVEL, D1        ;
        CMP.B  #22,D1           ; Check if player finished the game
        BEQ    gamecompletescr  ;      

        MULU   #SNK_SCR_SIZE,D1 ;offset=level*SNK_SCR_SIZE
        ADD.L  #STR_LEVEL1,D1   ;level=STR_LEVEL1+offset
        MOVEA.L D1,A2
        BSR     _LOADMEM     ; Load Level 1

        BSR     _DISPSCORE   ; Display Score

        MOVE.B LEVEL, D4     ; Display Level
        ADDI.B #1,D4
        MOVE.B #45,D1
        MOVE.B #24,D2
        BSR    _GOTOXY
        MOVE.L #STR_COL_YELLOW,A1 ; Set color
        BSR _DISPSTR 
        BSR    _DISPNUM10

        MOVE.B LIVES, D4     ; Display Lives
        MOVE.B #29,D1
        MOVE.B #24,D2
        BSR    _GOTOXY
        MOVE.L #STR_COL_YELLOW,A1 ; Set color
        BSR _DISPSTR 
        BSR    _DISPNUM10
        MOVE.B POS_X,D1
        MOVE.B POS_Y,D2        
        ADDI.B #1,D1 
        ADDI.B #1,D2
        BSR _GOTOXY ; goto worm starting point

start
        TRAP	#15 ; check if we have a character waiting for us
	  DC.W	4
	  BEQ movesnk; if not, move on  
        BSR _SGETCH  ; else, get the character

        ; check for WASD
        CMP.B  #'w',D0 ; check if up arrow was pressed
        BEQ    key_up
         
        CMP.B  #'s',D0 ; check if down arrow was pressed
        BEQ    key_down

        CMP.B  #'a',D0 ; check if left arrow was pressed
        BEQ    key_left

        CMP.B  #'d',D0 ; check if right arrow was pressed
        BEQ    key_right

        ; check for KEYPAD
        CMP.B  #'8',D0 ; check if up arrow was pressed
        BEQ    key_up
         
        CMP.B  #'5',D0 ; check if down arrow was pressed
        BEQ    key_down

        CMP.B  #'4',D0 ; check if left arrow was pressed
        BEQ    key_left

        CMP.B  #'6',D0 ; check if right arrow was pressed
        BEQ    key_right


        BRA    movesnk; if no key was pressed that means anything move on

key_up  
        CMP.B  #DIR_DOWN, LAST_DIR  ; don't let the snake move in oppose direction,
        BEQ    movesnk               ; since this = instant death

        MOVE.B #1, MOVING ; indicate we're moving if we haven't already
        MOVE.B #DIR_UP, DIRECTION ; set movement direction up
        BRA movesnk

key_down
        CMP.B  #DIR_UP, LAST_DIR  ; don't let the snake move in oppose direction,
        BEQ    movesnk               ; since this = instant death

        MOVE.B #1, MOVING ; indicate we're moving if we haven't already
        MOVE.B #DIR_DOWN, DIRECTION ; set movement direction down
        BRA movesnk

key_left
        CMP.B  #DIR_RIGHT, LAST_DIR  ; don't let the snake move in oppose direction,
        BEQ    movesnk               ; since this = instant death

        MOVE.B #1, MOVING ; indicate we're moving if we haven't already
        MOVE.B #DIR_LEFT, DIRECTION ; set movement direction left
        BRA movesnk

key_right 
        CMP.B  #DIR_LEFT, LAST_DIR  ; don't let the snake move in oppose direction,
        BEQ    movesnk               ; since this = instant death

        MOVE.B #1, MOVING ; indicate we're moving if we haven't already
        MOVE.B #DIR_RIGHT, DIRECTION ; set movement direction right
        BRA movesnk


movesnk CMP.B #0, MOVING  ; check if snake is moving
        BEQ incrand       ; if not, only thing to do is increase random counter, and repeat

        MOVE.L TIMER,D0
        ADDI.L #1, D0
        MOVE.L D0, TIMER

        CMP.L  SNK_SPEED,D0   ; Compare Timer with movement speed
        BNE    incrand      ; if timer is not up, restart
                           ; else, timer is up, move the snake
        MOVE.B DIRECTION, LAST_DIR
        CMP.B #DIR_UP, DIRECTION
        BEQ    mv_up

        CMP.B #DIR_DOWN, DIRECTION
        BEQ    mv_down

        CMP.B #DIR_LEFT, DIRECTION
        BEQ    mv_left

        CMP.B #DIR_RIGHT, DIRECTION
        BEQ    mv_right

mv_up   
        SUBI.B #1, POS_Y ; move snake up
        MOVE.L #0, TIMER ; reset timer
        BRA draw_snk
mv_down 
        ADDI.B #1, POS_Y ; move snake down
        MOVE.L #0, TIMER ; reset timer
        BRA draw_snk
mv_left 
        SUBI.B #1, POS_X ; move snake left
        MOVE.L #0, TIMER ; reset timer
        BRA draw_snk
mv_right
        ADDI.B #1, POS_X ; move snake right
        MOVE.L #0, TIMER ; reset timer
        BRA draw_snk
                        
draw_snk
        CMP.B  #$FF,POS_X  ; check if we have colided with the wall
        BEQ    gameover
        CMP.B  #78,POS_X
        BEQ    gameover
        CMP.B  #$FF,POS_Y
        BEQ    gameover
        CMP.B  #23,POS_Y
        BEQ    gameover

        BSR _DECAYSNK    ; decay snake
        BSR _DRAWSNK     ; draw new snake location


        CMP.B  #0, FOOD_AVAIL  
        BEQ    setfood          ; if not, set food in free spot
incrand
        BSR    _INCRAND     ; inc random x,y
        BRA    start  ; While game is in session
setfood     
        BSR    _DRAWFOOD          

        CMP.B   #$0A,FOOD_NUM  ;check what the food number is
        BNE     start          ;if less than 9, continue as normal
                               ;else player just leveled up
        MOVE.B  #0,FOOD_NUM    ;reset food num
        ADDI.B  #1,LEVEL       ;increase level
        BRA     loadlevel      ;load new level


gameover 
        CMP.B   #0,LIVES       ;check to see if we have any lives left
        BEQ     gameoverscr    ;if not, goto game over screen
        SUBI.B  #1,LIVES       ;else, subtract 1 from lives and restart current level
        BRA     loadlevel

gameoverscr
        MOVE #STR_GAME_OVER_SCR,A1 
        BSR _DISPSTR         
        BSR _SELECT_GAMEOVER ; game over menu
        BRA intro  ; loop to start if user selected play again  
        
gamecompletescr
        MOVE #STR_GAME_COMPLETE_SCR,A1 
        BSR _DISPSTR         
        TRAP #11        ; end game
        DC.W 0

  
; << End of Main >>


*************************************************************** 
; Function _INCRND
; Purpose  Increment Random X,Y position
***************************************************************

_INCRAND
        ADDI.W  #1,RAND_MEM
        CMP.W   #SNK_SCR_SIZE,RAND_MEM
        BNE     incdone
        MOVE.W  #$0,RAND_MEM

incdone
        RTS

*************************************************************** 
; Function _DRAWFOOD
; Purpose  Draw new food segment on the screen and in memory
***************************************************************

_DRAWFOOD
        MOVEM.L D0-D2/A0-A1,-(SP)        
        MOVE.L  #SNK_SCR,A0  ; load base address
        
findfoodspot
        BSR     _INCRAND
        MOVE.W  RAND_MEM,D2  
        MOVE.W  RAND_MEM,D0  
        MULU    #2,D0     
        CMP.W   #0,$00(A0,D0.W) ; check to see if this spot is clear
        BNE     findfoodspot    ; if not, find a new spot
        

        DIVU    #ARENA_X,D2     ; y = offset/x_max
        MOVE.L  D2,D1           ; x = remainder
        LSR.L   #8,D1
        LSR.L   #8,D1

        ADDI.B  #1,D1
        ADDI.B  #1,D2

        BSR _GOTOXY         ; goto snake segment
        MOVE.L #STR_COL_YELLOW,A1 
        BSR _DISPSTR 
        
        MOVE.B  FOOD_NUM,D0
        ADDI.B  #$30,D0
        BSR     _SPUTCH


        BSR _MARKFOOD        ; mark current position in memory
        MOVE.B #1,FOOD_AVAIL ; indicate food is set
        MOVEM.L	(SP)+,D0-D2/A0-A1
        RTS

*************************************************************** 
; Function _DRAWSNK
; Purpose  Draw new snake segment on the screen and in memory
***************************************************************

_DRAWSNK
        MOVEM.L D0-D1/A1,-(SP)
        MOVE.B POS_X,D1 
        MOVE.B POS_Y,D2
        ADDI.B #1,D1
        ADDI.B #1,D2
        BSR _GOTOXY         ; goto snake segment

        MOVE.L #STR_SNK_SEG,A1 
        BSR _DISPSTR 

        MOVE.L #STR_REV,A1 
        BSR _DISPSTR        ; go back a char to hide cursor    
        BSR _MARKSNK        ; mark current position in memory
        MOVEM.L	(SP)+,D0-D1/A1
        RTS

*************************************************************** 
; Function _CLRSNK
; Purpose  Clear snake segment on the screen 
***************************************************************

_CLRSNK
        MOVEM.L D0-D3,-(SP)
        ADDI.B  #1,D1
        ADDI.B  #1,D2
        BSR    _GOTOXY         ; goto snake segment
        MOVE.B #$20,D0      ; $20 = space
        BSR _SPUTCH         ; clear snake tile                
        MOVEM.L	(SP)+,D0-D3
        RTS

*************************************************************** 
; Function _MARKFOOD
; Purpose  Marks food position in memory
***************************************************************

_MARKFOOD
        MOVEM.L D0/A0,-(SP)
        MOVE.L #SNK_SCR,A0  ; load base address
        MOVE.W  RAND_MEM,D0     ; load random memory offset        
        MULU    #2,D0    ; x2 for word size 

        MOVE.W  #FOOD_LIFE,$00(A0,D0.W) ; mark spot = base+offset <- food
        MOVEM.L (SP)+,D0/A0
        RTS

*************************************************************** 
; Function _MARKSNK
; Purpose  Marks snake position in memory
;          memory location = base + x + (y*78)
;          memory location = SNK_SCR + POS_X + (POS_Y*78)
***************************************************************

_MARKSNK 
        MOVEM.L D0-D2/A0,-(SP)
        MOVE.L #SNK_SCR,A0 ; load base address
        CLR.L     D0
        CLR.L     D1

        MOVE.B  POS_Y,D0   
        MULU    #ARENA_X,D0     ; offset = y*78

        MOVE.B  POS_X,D1   
        ADD.W   D1,D0      ; offset = offset + x
        MULU    #2,D0     ; x2 for word size

        MOVE.W  $00(A0,D0.W),D2
        CMP.W   #FOOD_LIFE,D2    ; check if we're moving onto food
        BEQ     markfood         ; if not set, snake segement life
        CMP.W   #0,D2            ; check if we're moving onto snake tail or a wall
        BEQ     marksnk          ; if not, mark snake here
        BRA     gameover         ; else, we ran into tail. game over


markfood
        ADDI.B  #FOOD_GROWTH,DELAY_DECAY    ; else delay snake decay by DELAY_DECAY (aka eaten food)   
        ADDI.B  #1,SCORE 
        ADD.B   #1,FOOD_NUM      ; add one to food number       
        
        MOVE.B  #0,FOOD_AVAIL     ; set no food available
        BSR     _DISPSCORE   ; update score
        
marksnk
        CMP.B   #0,DELAY_DECAY   ; check to see if snake is growing
        BEQ     marknog          ; if not, set normal life 
        ADD.W   #1,SNK_LIFE
        MOVE.W  SNK_LIFE,D1  ; add 1 to snake life

        MOVE.W  D1,$00(A0,D0.W) ; mark spot = base+offset <- snake life
        MOVEM.L (SP)+,D0-D2/A0
        RTS

marknog MOVE.W  SNK_LIFE,$00(A0,D0.W) ; mark spot = base+offset <- snake life
        MOVEM.L (SP)+,D0-D2/A0
        RTS
   
*************************************************************** 
; Function _DECAYSNK
; Purpose  Decay entire snake in memory
***************************************************************

_DECAYSNK 
        MOVEM.L D0-D5/A0,-(SP)

        CMP.B   #0,DELAY_DECAY  ; Check if snake is growing
        BNE     decaydelay      ; if so, don't decay

        MOVE.L  #SNK_SCR,A0 ; load base address
        CLR.L     D0
        CLR.L     D1

        MOVE.L  #0,D1  ; current x (relative memory array)
        MOVE.L  #0,D2  ; current y (relative memory array)

nextseg
        MOVE.L  D2,D0    ; y -> d0
        MULU    #ARENA_X,D0  ; offset = y*78         
        ADD.W   D1,D0    ; offset = offset + x          

        MOVE.W  D0,D5    ; calc mem location
        MULU    #2,D5    ; x2 for word size


        CMP.W   #SNK_SCR_SIZE,D0  ; check to see if we're done 
        BNE     checkseg
        BRA     decaydelay

checkseg
        MOVE.W  $00(A0,D5.W),D4  ; move current life of segment x,y into D4
        CMP.W   #FOOD_LIFE,D4  ; check if food exists here
        BEQ     incseg  ; it does, don't decay it

        CMP.W   #0,D4    ; if snake segment exists here
        BNE     decayseg    ; decay segment here
        BRA     incseg      ; else no segment here, jump to inc segment code

decayseg         
        SUBI.W  #1,D4   ; Decay snake life by 1    
        MOVE.W  D4,$00(A0,D5.W) ; Update segment in memory
        CMP.W   #0,D4   ; check if life turned to 0
        BNE     incseg   ; if not 0 our segment still alive, jump to inc segment code
        BSR     _CLRSNK     ; else segment just died, update the screen (delete segment)


incseg  
        ADDI.B  #1,D1   ; add 1 to x
        CMP.B   #ARENA_X,D1  ; check to see if x is greater than max length (78)
        BEQ     incy
        BRA     nextseg    ; if not start checking next segment
incy    MOVE.B  #0,D1   ; x = 1
        ADDI.B  #1,D2   ; add 1 to y
        BRA     nextseg    ; check next segment


decaydelay
        CMP.B   #0,DELAY_DECAY  
        BEQ     donedecay
        SUBI.B  #1,DELAY_DECAY
        CMP.B   #0,DELAY_DECAY  ; we have just ended a decay period
        BNE     donedecay
        
donedecay
        MOVEM.L (SP)+,D0-D5/A0
        RTS


*************************************************************** 
; Function _LOADMEM
; Purpose  Clear a bitmap to the snake space in memory (load level)
; Param    A2 - Bitmap in memory
***************************************************************

_LOADMEM
        MOVEM.L D0-D2/A0,-(SP)
        MOVE.L  #0,D0
        MOVE.L  #SNK_SCR,A0 ; load base address
loadnextmem
        MOVE.W  D0,D1
        MULU    #2,D1    ; x2 for word size
        MOVE.B  $00(A2,D0.W),D2
        CMP.B   #'w',D2      ;check if segment is a wall
        BEQ     loadwallseg       
        BRA     loadinc 
loadwallseg
        MOVE.W  #WALL_LIFE,$00(A0,D1.W)  ; move current life of segment x,y into D4
        MOVE.L  D0,D2
        DIVU    #ARENA_X,D2     ; y = offset/x_max
        MOVE.L  D2,D1           ; x = remainder
        LSR.L   #8,D1
        LSR.L   #8,D1

        ADDI.B  #1,D1
        ADDI.B  #1,D2

        BSR _GOTOXY         ; goto snake segment
        MOVE.L #STR_WALL,A1 
        BSR _DISPSTR         

loadinc
        ADDI.W  #1,D0       
        CMP.W   #SNK_SCR_SIZE,D0
        BNE     loadnextmem  
      
        MOVEM.L (SP)+,D0-D2/A0
        RTS

*************************************************************** 
; Function _CLEARMEM
; Purpose  Clear entire snake space in memory
***************************************************************

_CLEARMEM
        MOVEM.L D0-D1/A0,-(SP)
        MOVE.L  #0,D0
        MOVE.L  #SNK_SCR,A0 ; load base address
clearnextmem
        MOVE.W    D0,D1
        MULU    #2,D1    ; x2 for word size
        MOVE.W  #$0000,$00(A0,D1.W)  ; move current life of segment x,y into D4

        ADDI.W  #1,D0
        CMP.W   #SNK_SCR_SIZE,D0
        BNE     clearnextmem  
      
        MOVEM.L (SP)+,D0-D1/A0
        RTS

        
*************************************************************** 
; Function _DISPSCORE
; Purpose  Display a score to the screen
***************************************************************


_DISPSCORE       
        MOVEM.L D0-D4/A1,-(SP)

        MOVE    #13,D1    ; x Goto word "Score:" on the screen
        MOVE    #24,D2    ; y
        BSR     _GOTOXY
        MOVE.L #STR_COL_YELLOW,A1 
        BSR _DISPSTR          

        MOVE.B SCORE,D4
        BSR    _DISPNUM10


        MOVEM.L (SP)+,D0-D4/A1
        RTS        

*************************************************************** 
; Function _DISPNUM10
; Purpose  Display a number to the screen in BASE 10
***************************************************************

_DISPNUM10
        MOVEM.L D0-D4/A1,-(SP)
        CLR.B   D3        
        CMP.B   #0,D4     ; if it's zero to start with, we don't need to count
        BEQ     scoreskipten

scorecnt                     ; count by 10 algorithm
        ADDI.B  #1,D3        ; add 1 to base 10 counter
        SUBI.B  #1,D4        ; decrement actual counter
        MOVE.B  D3,D0        
        ANDI.B  #$0F,D0      ; check if we have 10 in 1's column
        CMP.B   #$0A,D0
        BEQ     scoreaddten   ; if so, add 1 to 10's column, clear 1's 
        BRA     scoredonecnt  ; else see if we're done

scoreaddten
        ADDI.B  #$10,D3      ; Add 1 to tens
        ANDI.B  #$F0,D3      ; Clears ones

scoredonecnt
        CMP.B   #0,D4        ; if not done,
        BNE     scorecnt      ; start count algoritm again
        
        MOVE.B  D3,D0        ; display 10's digit
        LSR.B   #4,D0          
        CMP.B   #$0,D0       ; if 10's digit is a 0, don't output it
        BEQ     scoreskipten

        ADDI.B  #$30,D0
        BSR     _SPUTCH

scoreskipten
        MOVE.B  D3,D0        ; display 1's digit
        ANDI    #$0F,D0
        ADDI.B  #$30,D0
        BSR     _SPUTCH
        
        MOVEM.L (SP)+,D0-D4/A1
        RTS


*************************************************************** 
; Function _DISPSTR
; Purpose  Display a string in memory to the screen
***************************************************************


_DISPSTR
        MOVEM.L D0,-(SP)
nextch  
        MOVE.B  (A1)+,D0 ; get character from pointer
        CMP.B #$0,D0    ; see if it's null
        BEQ donech       ; if so, we're done
        BSR _SPUTCH      ; else display it
        BRA nextch
donech  
        MOVEM.L (SP)+,D0
        RTS

***************************************************************
; Function _GOTOXY
; Purpose  Goto position on the screen specified by D1(x),D2(y)
***************************************************************

_GOTOXY 
        MOVEM.L D0-D4,-(SP)
        
        MOVE.B  D2,D4   ; setup y for count
        CLR.W   D3

        MOVEA.L #STR_ESC,A1 
        BSR     _DISPSTR     ; send ANSI escape sequence

        BRA     gotocnt
gotocnty
        MOVE.B  #$FF,D2
        MOVE.B  D1,D4   ; setup x for count
        CLR.W   D3

        MOVE.B  #';',D0
        BSR     _SPUTCH
 
gotocnt                      ; count by 10 algorithm
        ADDI.B  #1,D3        ; add 1 to base 10 counter
        SUBI.B  #1,D4        ; decrement actual counter
        MOVE.B  D3,D0        
        ANDI.B  #$0F,D0      ; check if we have 10 in 1's column
        CMP.B   #$0A,D0
        BEQ     gotoaddten   ; if so, add 1 to 10's column, clear 1's 
        BRA     gotodonecnt  ; else see if we're done

gotoaddten
        ADDI.B  #$10,D3      ; Add 1 to tens
        ANDI.B  #$F0,D3      ; Clears ones

gotodonecnt
        CMP.B   #0,D4        ; if not done,
        BNE     gotocnt      ; start count algoritm again
        
        MOVE.B  D3,D0        ; display 10's digit
        LSR.B   #4,D0          
        CMP     #$0,D0       ; if 10's digit is a 0, don't output it
        BEQ     gotoskipten

        ADDI.B  #$30,D0
        BSR     _SPUTCH

gotoskipten
        MOVE.B  D3,D0        ; display 1's digit
        ANDI    #$0F,D0
        ADDI.B  #$30,D0
        BSR     _SPUTCH

        CMP.B   #$FF,D2      ; check to see if we already counted Y
        BNE     gotocnty     ; if not, start counting


        MOVE.B  #'H',D0      ; Finish ANSI control in form <ESC>[Yn:XnH
        BSR     _SPUTCH

        MOVEM.L	(SP)+,D0-D4
        RTS    

*************************************************************** 
; Function _SELECT_DIFF
; Purpose  Select Difficulty
***************************************************************

_SELECT_DIFF
        MOVEM.L D0-D3,-(SP)
        ; Select Difficulty
draw_select
        MOVE.B   #8,D1    
        MOVE.B   #14,D2
        CMP.B     #0,DIFFICULTY  ; 
        BEQ     draw_easy_select        
        CMP.B     #1,DIFFICULTY  ; 
        BEQ     draw_medium_select
        CMP.B     #2,DIFFICULTY  ; 
        BEQ     draw_hard_select
        CMP.B     #3,DIFFICULTY  ; 
        BEQ     draw_insane_select

draw_easy_select     ; Draw EASY selected
        MOVE.L  #SNK_SPEED_EASY,SNK_SPEED ; set snake speed for this option

        BSR      _GOTOXY
        MOVEA.L  #STR_SEASY,A1 ; draw selected
        BSR _DISPSTR
        ADDI.B   #1,D2         ; add 1 to y
        BSR      _GOTOXY
        MOVEA.L  #STR_MEDIUM,A1 ; draw unselected
        BSR _DISPSTR
        ADDI.B   #1,D2         ; add 1 to y
        BSR      _GOTOXY
        MOVEA.L  #STR_HARD,A1 ; draw unselected
        BSR _DISPSTR
        ADDI.B   #1,D2         ; add 1 to y
        BSR      _GOTOXY
        MOVEA.L  #STR_INSANE,A1 ; draw unselected
        BSR _DISPSTR
        BRA get_select
draw_medium_select
        MOVE.L  #SNK_SPEED_MEDIUM,SNK_SPEED ; set snake speed for this option

        BSR      _GOTOXY
        MOVEA.L  #STR_EASY,A1 
        BSR _DISPSTR
        ADDI.B   #1,D2         ; add 1 to y
        BSR      _GOTOXY
        MOVEA.L  #STR_SMEDIUM,A1 
        BSR _DISPSTR
        ADDI.B   #1,D2         ; add 1 to y
        BSR      _GOTOXY
        MOVEA.L  #STR_HARD,A1 
        BSR _DISPSTR
        ADDI.B   #1,D2         ; add 1 to y
        BSR      _GOTOXY
        MOVEA.L  #STR_INSANE,A1
        BSR _DISPSTR
        BRA get_select
draw_hard_select
        MOVE.L  #SNK_SPEED_HARD,SNK_SPEED ; set snake speed for this option

        BSR      _GOTOXY
        MOVEA.L  #STR_EASY,A1 
        BSR _DISPSTR
        ADDI.B   #1,D2  
        BSR      _GOTOXY       ; add 1 to y
        MOVEA.L  #STR_MEDIUM,A1 
        BSR _DISPSTR
        ADDI.B   #1,D2         ; add 1 to y
        BSR      _GOTOXY
        MOVEA.L  #STR_SHARD,A1 
        BSR _DISPSTR
        ADDI.B   #1,D2         ; add 1 to y
        BSR      _GOTOXY
        MOVEA.L  #STR_INSANE,A1 
        BSR _DISPSTR
        BRA get_select
draw_insane_select
        MOVE.L  #SNK_SPEED_INSANE,SNK_SPEED ; set snake speed for this option

        BSR      _GOTOXY
        MOVEA.L  #STR_EASY,A1 ; draw splash screen
        BSR _DISPSTR
        ADDI.B   #1,D2         ; add 1 to y
        BSR      _GOTOXY
        MOVEA.L  #STR_MEDIUM,A1 ; draw splash screen
        BSR _DISPSTR
        ADDI.B   #1,D2         ; add 1 to y
        BSR      _GOTOXY
        MOVEA.L  #STR_HARD,A1 ; draw splash screen
        BSR _DISPSTR
        ADDI.B   #1,D2         ; add 1 to y
        BSR      _GOTOXY
        MOVEA.L  #STR_SINSANE,A1 ; draw splash screen
        BSR _DISPSTR
        BRA get_select

get_select
        MOVE.B   #0,D1    
        MOVE.B   #0,D2
        BSR _SGETCH  ; else, get the character 

        CMP.B  #'s',D0 ; check if down arrow was pressed
        BEQ    select_key_down

        CMP.B  #'w',D0 ; check if left arrow was pressed
        BEQ    select_key_up  
       
        CMP.B  #$0D,D0 ; check if enter key was pressed
        BNE    draw_select  
        MOVEM.L (SP)+,D0-D3
        RTS

select_key_down
        CMP.B   #3,DIFFICULTY       ; check to see if we're at bottom of menu
        BEQ     draw_select  ; if so, don't go down any more
        ADDI.B  #1,DIFFICULTY
        BRA     draw_select

select_key_up
        CMP.B   #0,DIFFICULTY       ; check to see if we're at top of menu
        BEQ     draw_select  ; if so, don't go up any more
        SUBI.B   #1,DIFFICULTY
        BRA      draw_select

*************************************************************** 
; Function _SELECT_GAMEOVER
; Purpose  Game over selection screen
***************************************************************

_SELECT_GAMEOVER
        MOVEM.L D0-D3,-(SP)
        ; Select Play Again or End Game
        MOVE.B  #0,D3  ; set default select to Medium

gg_draw_select
        CMP     #0,D3  
        BEQ     gg_draw_yes_select        
        CMP     #1,D3  
        BEQ     gg_draw_no_select

gg_draw_yes_select     ; Draw EASY selected
        MOVE.B   #23,D1    
        MOVE.B   #19,D2
        BSR      _GOTOXY
        MOVEA.L  #STR_GG_SYES,A1 ; draw Yes option selected screen
        BSR _DISPSTR
        MOVE.B   #44,D1    
        MOVE.B   #19,D2
        BSR      _GOTOXY
        MOVEA.L  #STR_GG_NO,A1 ; draw No option deselected screen
        BSR _DISPSTR
        BRA gg_get_select

gg_draw_no_select
        MOVE.B   #23,D1    
        MOVE.B   #19,D2
        BSR      _GOTOXY
        MOVEA.L  #STR_GG_YES,A1 ;  draw Yes option deselected screen
        BSR _DISPSTR
        MOVE.B   #44,D1    
        MOVE.B   #19,D2
        BSR      _GOTOXY
        MOVEA.L  #STR_GG_SNO,A1 ; draw No option selected screen

        BSR _DISPSTR

gg_get_select
        MOVE.B   #0,D1    
        MOVE.B   #0,D2
        BSR      _GOTOXY
        BSR _SGETCH  ; else, get the character 

        CMP.B  #'a',D0 ; check if down arrow was pressed
        BEQ    gg_select_key_left

        CMP.B  #'d',D0 ; check if left arrow was pressed
        BEQ    gg_select_key_right
       
        CMP.B  #$0D,D0 ; check if enter key was pressed
        BNE    gg_get_select  ; if not, get another key

        CMP.B  #0,D3   ; see if we're done
        BEQ    gg_done_select 

        TRAP    #11    ; end game
        DC.W    0

        

gg_done_select
        MOVEM.L (SP)+,D0-D3
        RTS

gg_select_key_left
        CMP.B   #0,D3       ; check to see if we're at top of menu
        BEQ     gg_select_key_right  ; if so, don't go up any more
        SUBI.B  #1,D3
        BRA     gg_draw_select

gg_select_key_right
        CMP.B   #1,D3       ; check to see if we're at bottom of menu
        BEQ     gg_select_key_left  ; if so, don't go down any more
        ADDI.B  #1,D3
        BRA     gg_draw_select




***** Subroutine for receiving a character - simulator *****

_SGETCH 
        TRAP	#15
        DC.W	3
        RTS

**** Subroutine for sending character - simulator *****

_SPUTCH 
        TRAP	#15         
        DC.W	1
        RTS

***** Subroutine for receiving a character - trainer board *****
	  
_GETCH  
        MOVEM.L	D1/A0,-(SP)
	  LEA	DUART,A0
IP_POLL 
        MOVE.B	SRA(A0),D1
        BTST	#RxRDY,D1
        BEQ	IP_POLL
        MOVE.B	RBA(A0),D0
        MOVEM.L	(SP)+,D1/A0
        RTS

***** Subroutine for sending character - trainer board *****

_PUTCH  
        MOVEM.L	D1/A0,-(SP)
	  LEA	DUART,A0
OP_POLL 
        MOVE.B	SRA(A0),D1
        BTST	#TxRDY,D1
        BEQ	OP_POLL
        MOVE.B	D0,TBA(A0)
        MOVEM.L	(SP)+,D1/A0
        RTS

LIVES   DC.B $00
LEVEL   DC.B $00
FOOD_NUM   DC.B $00
SNK_SPEED DC.L $00000000
SNK_LIFE  DC.W $00
DELAY_DECAY DC.B $00
FOOD_AVAIL DC.B  $00
RAND_MEM  DC.W  $0000

SCORE     DC.B  $00

DIFFICULTY DC.B  $00
POS_X     DC.B  $00
POS_Y     DC.B  $00
LAST_DIR  DC.B  $0F
DIRECTION DS.B  1
MOVING    DS.B  1
TIMER     DS.L  1

SNK_SCR   DS.W  SNK_SCR_SIZE  ; x*y = 78*22

STR_EASY   DC.B    '[37;40m [32;40mEASY[37;40m',0
STR_MEDIUM DC.B    '[32;40mMEDIUM[37;40m',0
STR_HARD   DC.B    '[37;40m [32;40mHARD[37;40m',0
STR_INSANE DC.B    '[32;40mINSANE[37;40m',0

STR_SEASY   DC.B   '[37;40m [30;42mEASY[37;40m',0
STR_SMEDIUM DC.B   '[30;42mMEDIUM[37;40m',0
STR_SHARD   DC.B   '[37;40m [30;42mHARD[37;40m',0
STR_SINSANE DC.B   '[30;42mINSANE[37;40m',0

STR_GG_YES  DC.B   '[34;40mPLAY AGAIN[37;40m',0
STR_GG_SYES DC.B   '[30;44mPLAY AGAIN[37;40m',0

STR_GG_NO   DC.B   '[34;40mNO, THANKS[37;40m',0
STR_GG_SNO  DC.B   '[30;44mNO, THANKS[37;40m',0

STR_ESC   DC.B    $1B,'[',0
STR_UP    DC.B    $1B,'[1A',0
STR_DOWN  DC.B    $1B,'[1B',0
STR_FWD   DC.B    $1B,'[1C',0
STR_REV   DC.B    $1B,'[1D',0
STR_CLS   DC.B    $1B,'[2J',0
STR_COL_RED   DC.B  $1B,'[31;40m',0
STR_COL_YELLOW DC.B '[33;40m',0
STR_COL_GREEN  DC.B '[32;40m',0

STR_SNK_SEG DC.B     $1B,'[32;40m',$DB,0
STR_WALL  DC.B     $1B,'[31;40m','˛',0
STR_FOOD  DC.B     $1B,'[33;40m','“',0
STR_HOME  DC.B    '[0;0H',0

STR_SPLASH_SCR DC.B  '[2J[0m[5C[1;32m   [0m[39C[1;32m [0m',$0D,$0A
 DC.B '[12C[1;31m≤≤≤[37mª[0m[3C[1;31m≤≤[37mª [31m≤≤[37mª [31m≤≤≤≤≤≤[37mª  [31m≤≤≤≤≤≤[37mª  [31m≤≤[37mª[0m[6C[1;31m≤≤≤≤≤≤≤[37mª [31m≤≤≤≤≤≤≤[37mª[0m',$0D,$0A
 DC.B '[12C[1;31m€€€€[37mª  [31m€€[37m∫ [31m€€[37m∫ [31m€€[37m…ÕÕ[31m€€[37mª [31m€€[37m…ÕÕ[31m€€[37mª [31m€€[37m∫[0m[6C[1;31m€€[37m…ÕÕÕÕº [31m€€[37m…ÕÕÕÕº[0m',$0D,$0A
 DC.B '[12C[1;31m≤≤[37m…[31m≤≤[37mª [31m≤≤[37m∫ [31m≤≤[37m∫ [31m≤≤≤≤≤≤[37m…º [31m≤≤≤≤≤≤[37m…º [31m≤≤[37m∫[0m[6C[1;31m≤≤≤≤≤[37mª[0m[3C[1;31m≤≤≤≤≤≤≤[37mª[0m',$0D,$0A
 DC.B '[12C[1;31m±±[37m∫»[31m±±[37mª[31m±±[37m∫ [31m±±[37m∫ [31m±±[37m…ÕÕ[31m±±[37mª [31m±±[37m…ÕÕ[31m±±[37mª [31m±±[37m∫[0m[6C[1;31m±±[37m…ÕÕº[0m[3C[1m»ÕÕÕÕ[31m±±[37m∫[0m',$0D,$0A
 DC.B '[3C[1;32m        [0m [1;31m∞∞[37m∫ »[31m∞∞∞∞[37m∫ [31m∞∞[37m∫ [31m∞∞∞∞∞∞[37m…º [31m∞∞∞∞∞∞[37m…º [31m∞∞∞∞∞∞∞[37mª [31m∞∞∞∞∞∞∞[37mª [31m∞∞∞∞∞∞∞[37m∫[0m',$0D,$0A
 DC.B '[12C[1m»Õº  »ÕÕÕº »Õº »ÕÕÕÕÕº  »ÕÕÕÕÕº  »ÕÕÕÕÕÕº »ÕÕÕÕÕÕº »ÕÕÕÕÕÕº[0m',$0D,$0A
 DC.B '[32C[1;33m⁄ƒƒƒƒø  ⁄ƒƒƒƒƒø ⁄ƒƒƒƒƒø ⁄ƒƒƒƒƒø ⁄ƒƒƒƒƒø[0m',$0D,$0A
 DC.B '[32C[1;33m≥ ⁄ƒƒŸ  ≥ ⁄ƒø ≥ ≥ ⁄ƒø ≥ ≥ ⁄ƒø ≥ ≥ ⁄ƒø ≥[0m',$0D,$0A
 DC.B '[25C[1;32m€€€    [33m≥ ¿ƒƒƒø ≥ ¿ƒŸ ≥ ≥ ≥ ≥ ≥ ≥ ≥ ≥ ≥ ≥ ≥ ≥ ≥[0m',$0D,$0A
 DC.B '[22C[1;32m€€€  €€   [33m≥ ⁄ƒø ≥ ≥ ⁄ƒø ≥ ≥ ≥ ≥ ≥ ≥ ≥ ≥ ≥ ≥ ≥ ≥ ≥[0m',$0D,$0A
 DC.B '[22C[1;32m€€€€€€€   [33m≥ ¿ƒŸ ≥ ≥ ¿ƒŸ ≥ ≥ ¿ƒŸ ≥ ≥ ¿ƒŸ ≥ ≥ ¿ƒŸ ≥[0m',$0D,$0A
 DC.B '[4C[37m’Õ[33mDifficulty[37mÕ∏[0m  [1m [0m [1;32mﬂﬂﬂﬂﬂﬂ€€  [33m¿ƒƒƒƒƒŸ ¿ƒƒƒƒƒŸ ¿ƒƒƒƒƒŸ ¿ƒƒƒƒƒŸ ¿ƒƒƒƒƒŸ[0m',$0D,$0A
 DC.B '[4C[37m≥[12C≥[0m[10C[1;32m€€[0m[11C[1;32m€€€€[0m',$0D,$0A
 DC.B '[4C[37m≥ [0m[3C[1;32mEasy[0m[4C[1m≥[0m[9C[1;32m€€[0m[11C[1;32m€€  €€[0m[14C[1m [31mMovement Keys   [32m  [0m',$0D,$0A
 DC.B '[4C[37m≥ [32m  Medium[0m[3C[1m≥[0m[8C[1;32m€€[0m[11C[1;32m€€    €€[0m[6C[1;32m [0m [1;32m‹[0m',$0D,$0A
 DC.B '[4C[37m≥ [0m[3C[1;32mHard[0m[4C[1m≥[0m[8C[1;32m€€[0m[10C[1;32m€€[0m[5C[1;32m€€[0m[5C[1;32m‹ﬂ[0m[6C[1m [5C[7C ',$0D,$0A
 DC.B '[0m[4C[1m≥ [32m  Insane[0m[3C[1m≥[0m[8C[1;32m€€[0m[10C[1;32m€€    €€[0m[5C[1;32m‹ﬂ[0m[7C[1m    [37m⁄ƒƒƒø',$0D,$0A
 DC.B '[0m[4C[1m≥        [4C≥[0m[9C[1;32m€€[0m[8C[1;32m€€[0m[5C[1;32m€€[0m[4C[1;32m‹€[0m[8C[1m    [37m≥ W ≥',$0D,$0A
 DC.B '[0m[4C[1m‘ÕÕÕÕÕÕÕÕÕÕÕÕæ[0m [9C[1;32m€€[0m[6C[1;32m€€[0m[6C[1;32m€€‹  €€[0m[9C[1m [37m⁄ƒƒ¡¬ƒƒ¡¬ƒƒƒø',$0D,$0A
 DC.B '[0m[19C  [8C[1;32m€€€€€€€€[0m[8C[1;32m€€€€€[0m[9C[1m[37m  ≥ A ≥ S ≥ D ≥ ',$0D,$0A
 DC.B '[0m[5C[1;32m [0m                  [36C[1m[37m ¿ƒƒƒ¡ƒƒƒ¡ƒƒƒŸ',$0D,$0A
 DC.B '[30m [34mwww.syndsys.com[0m      [1m                                                    ',$0D,$0A
 DC.B ' Programmed By Josh Henn[0m',$0D,$0A,0


STR_ARENA_SCR DC.B '[2J[0m[1;31m…ÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕª∫[0m[78C[1;31m∫∫ '
 DC.B '[0m[77C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫'
 DC.B '[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫'
 DC.B '[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫[0m[78C[1;31m∫∫'
 DC.B '                                                                              ∫»ÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕ'
 DC.B 'ÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕº      [33;40mSCORE:          LIVES:          LEVEL:[33;40m                    www.syndsys.com[0m',0

STR_GAME_OVER_SCR DC.B '[2J[0m',$0D,$0A
 DC.B '',$0D,$0A
 DC.B '',$0D,$0A
 DC.B '',$0D,$0A
 DC.B '[16C[1;31m‹€€€€€€€€‹  ‹€€€€€€€‹  ‹€€‹   ‹€€‹ ‹€€€€€€€€€‹[0m',$0D,$0A
 DC.B '[15C[1;31m€€€€ﬂ   ﬂﬂﬂ €€€€ﬂﬂﬂ€€€€ €€€€€‹€€€€€ €€€€    ﬂﬂﬂ[0m',$0D,$0A
 DC.B '[15C[1;31m€€€€  ‹‹‹‹  €€€€‹‹‹€€€€ €€€€€€€€€€€ €€€€‹‹‹‹‹[0m',$0D,$0A
 DC.B '[15C[1;31m€€€€  ﬂﬂ€€€ €€€€€€€€€€€ €€€€ ﬂ €€€€ €€€€ﬂﬂﬂﬂﬂ[0m',$0D,$0A
 DC.B '[15C[1;31m€€€€€‹‹‹€€€ €€€€   €€€€ €€€€   €€€€ €€€€    ‹‹‹[0m',$0D,$0A
 DC.B '[16C[1;31mﬂ€€€€€€€€ﬂ ﬂ€€ﬂ   ﬂ€€ﬂ ﬂ€€ﬂ   ﬂ€€ﬂ ﬂ€€€€€€€€€ﬂ[0m',$0D,$0A
 DC.B '',$0D,$0A
 DC.B '[16C[1;31m‹€€€€€€€‹  ‹€€‹   ‹€€‹ ‹€€€€€€€€€‹ ‹€€€€€€€€‹[0m',$0D,$0A
 DC.B '[15C[1;31m€€€€ﬂﬂﬂ€€€€ €€€€   €€€€ €€€€    ﬂﬂﬂ €€€ﬂ   ﬂ€€€[0m',$0D,$0A
 DC.B '[15C[1;31m€€€[0m[5C[1;31m€€€ €€€€   €€€€ €€€€‹‹‹‹‹   €€€‹   ‹€€€[0m',$0D,$0A
 DC.B '[15C[1;31m€€€[0m[5C[1;31m€€€ ﬂ€€€‹ ‹€€€ﬂ €€€€ﬂﬂﬂﬂﬂ   €€€€€€€€€ﬂ[0m',$0D,$0A
 DC.B '[15C[1;31m€€€€‹‹‹€€€€  ﬂ€€€€€€€ﬂ  €€€€    ‹‹‹ €€€€ ﬂ€€€€‹[0m',$0D,$0A
 DC.B '[16C[1;31mﬂ€€€€€€€ﬂ[0m[5C[1;31mﬂ€€€ﬂ    ﬂ€€€€€€€€€ﬂ ﬂ€€ﬂ   ﬂ€€ﬂ[0m',$0D,$0A
 DC.B '',$0D,$0A
 DC.B '[15C[1;34m’ÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕ∏[0m',$0D,$0A
 DC.B '[15C[1;34m≥[0m [1;34m     [0m [1;34mPLAY AGAIN    [0m[7C[1;34mNO, THANKS[0m[7C[1;34m≥[0m',$0D,$0A
 DC.B '[15C[1;34m‘ÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕÕæ[0m',$0D,$0A,0

 
STR_GAME_COMPLETE_SCR DC.B '[2J[0m',$0D,$0A
 DC.B '',$0D,$0A
 DC.B '',$0D,$0A
 DC.B '',$0D,$0A
 DC.B '',$0D,$0A
 DC.B '[23C[1;34mﬂﬂﬂﬂﬂ    ﬂﬂﬂﬂﬂ  ﬂﬂﬂ   ﬂﬂﬂ ﬂﬂﬂﬂﬂﬂﬂ[0m',$0D,$0A
 DC.B '[22C[1;34mﬂﬂ[0m[7C[1;34mﬂﬂ   ﬂﬂ ﬂﬂﬂﬂ ﬂﬂﬂﬂ ﬂﬂ[0m',$0D,$0A
 DC.B '[21C[1;34mﬂﬂﬂ  ﬂﬂﬂﬂ ﬂﬂﬂﬂﬂﬂﬂ ﬂﬂ ﬂﬂﬂ ﬂﬂ ﬂﬂﬂﬂﬂ[0m',$0D,$0A
 DC.B '[22C[1;34mﬂﬂ   ﬂﬂ  ﬂﬂ   ﬂﬂ ﬂﬂ  ﬂ  ﬂﬂ ﬂﬂ[0m',$0D,$0A
 DC.B '[23C[1;34mﬂﬂﬂﬂﬂ   ﬂﬂ   ﬂﬂ ﬂﬂ[0m[5C[1;34mﬂﬂ ﬂﬂﬂﬂﬂﬂﬂ[0m',$0D,$0A
 DC.B '',$0D,$0A
 DC.B '[7C[1;34mﬂﬂﬂﬂﬂ    ﬂﬂﬂﬂﬂ   ﬂﬂﬂ   ﬂﬂﬂ ﬂﬂﬂﬂﬂﬂ  ﬂﬂ[0m[6C[1;34mﬂﬂﬂﬂﬂﬂﬂ ﬂﬂﬂﬂﬂﬂﬂﬂ ﬂﬂﬂﬂﬂﬂﬂ[0m',$0D,$0A
 DC.B '[6C[1;34mﬂﬂ   ﬂﬂ  ﬂﬂ   ﬂﬂ  ﬂﬂﬂﬂ ﬂﬂﬂﬂ ﬂﬂ   ﬂﬂ ﬂﬂ[0m[6C[1;34mﬂﬂ[0m[9C[1;34mﬂﬂ    ﬂﬂ[0m',$0D,$0A
 DC.B '[5C[1;34mﬂﬂﬂ[0m[6C[1;34mﬂﬂﬂ   ﬂﬂﬂ ﬂﬂ ﬂﬂﬂ ﬂﬂ ﬂﬂﬂﬂﬂﬂ  ﬂﬂ[0m[6C[1;34mﬂﬂﬂﬂﬂ[0m[6C[1;34mﬂﬂ    ﬂﬂﬂﬂﬂ[0m',$0D,$0A
 DC.B '[6C[1;34mﬂﬂ   ﬂﬂ  ﬂﬂ   ﬂﬂ  ﬂﬂ  ﬂ  ﬂﬂ ﬂﬂ[0m[6C[1;34mﬂﬂ[0m[6C[1;34mﬂﬂ[0m[9C[1;34mﬂﬂ    ﬂﬂ[0m',$0D,$0A
 DC.B '[7C[1;34mﬂﬂﬂﬂﬂ    ﬂﬂﬂﬂﬂ   ﬂﬂ[0m[5C[1;34mﬂﬂ ﬂﬂ[0m[6C[1;34mﬂﬂﬂﬂﬂﬂﬂ ﬂﬂﬂﬂﬂﬂﬂ    ﬂﬂ    ﬂﬂﬂﬂﬂﬂﬂ[0m',$0D,$0A
 DC.B '',$0D,$0A
 DC.B '',$0D,$0A
 DC.B '[27C[1;31mProgrammed by Josh Henn[0m',$0D,$0A
 DC.B '[31C[1;31mwww.syndsys.com',$0D,$0A,0

STR_LEVEL1 
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '

STR_LEVEL2 
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '             wwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwww              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '

STR_LEVEL3 
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                      w                                       '
 DC.B '                                      w                                       '
 DC.B '                                      w                                       '
 DC.B '                                      w                                       '
 DC.B '                                      w                                       '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                 wwwwwwwwwwwwwww             wwwwwwwwwwwwwww                  '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                      w                                       '
 DC.B '                                      w                                       '
 DC.B '                                      w                                       '
 DC.B '                                      w                                       '
 DC.B '                                      w                                       '
 DC.B '                                      w                                       '
 DC.B '                                                                              '
 DC.B '                                                                              '

STR_LEVEL4 
 DC.B '              w                                                               '
 DC.B '              w                                                               '
 DC.B '              w                                                               '
 DC.B '              w                                                               '
 DC.B '              w                                                               '
 DC.B '              w                                wwwwwwwwwwwwwwwwwwwwwwwwwwwwwww'
 DC.B '              w                                                               '
 DC.B '              w                                                               '
 DC.B '              w                                                               '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                         w                    '
 DC.B '                                                         w                    '
 DC.B '                                                         w                    '
 DC.B 'wwwwwwwwwwwwwwwwwwwwwwwww                                w                    '
 DC.B '                                                         w                    '
 DC.B '                                                         w                    '
 DC.B '                                                         w                    '
 DC.B '                                                         w                    '
 DC.B '                                                         w                    '
 DC.B '                                                         w                    '


STR_LEVEL5 
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                      wwwwwwwwwwwwwwwwwwwwwwwwwwwwwww                         '
 DC.B '                                                                              '
 DC.B '                    w                                 w                       '
 DC.B '                    w                                 w                       '
 DC.B '                    w                                 w                       '
 DC.B '                    w                                 w                       '
 DC.B '                    w                                 w                       '
 DC.B '                    w                                 w                       '
 DC.B '                    w                                 w                       '
 DC.B '                    w                                 w                       '
 DC.B '                    w                                 w                       '
 DC.B '                    w                                 w                       '
 DC.B '                                                                              '
 DC.B '                      wwwwwwwwwwwwwwwwwwwwwwwwwwwwwww                         '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '

STR_LEVEL6 
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '

STR_LEVEL7 
 DC.B '                                      w                                       '
 DC.B '                                                                              '
 DC.B '                                      w                                       '
 DC.B '                                                                              '
 DC.B '                                      w                                       '
 DC.B '                                                                              '
 DC.B '                                      w                                       '
 DC.B '                                                                              '
 DC.B '                                      w                                       '
 DC.B '                                                                              '
 DC.B '                                      w                                       '
 DC.B '                                                                              '
 DC.B '                                      w                                       '
 DC.B '                                                                              '
 DC.B '                                      w                                       '
 DC.B '                                                                              '
 DC.B '                                      w                                       '
 DC.B '                                                                              '
 DC.B '                                      w                                       '
 DC.B '                                                                              '
 DC.B '                                      w                                       '
 DC.B '                                                                              '

STR_LEVEL8 
 DC.B '       w                    w                    w                  w         '
 DC.B '       w                    w                    w                  w         '
 DC.B '       w                    w                    w                  w         '
 DC.B '       w                    w                    w                  w         '
 DC.B '       w                    w                    w                  w         '
 DC.B '       w         w          w         w          w         w        w         '
 DC.B '       w         w          w         w          w         w        w         '
 DC.B '       w         w          w         w          w         w        w         '
 DC.B '       w         w          w         w          w         w        w         '
 DC.B '       w         w          w         w          w         w        w         '
 DC.B '       w         w          w         w          w         w        w         '
 DC.B '       w         w          w         w          w         w        w         '
 DC.B '       w         w          w         w          w         w        w         '
 DC.B '       w         w          w         w          w         w        w         '
 DC.B '       w         w          w         w          w         w        w         '
 DC.B '       w         w          w         w          w         w        w         '
 DC.B '       w         w          w         w          w         w        w         '
 DC.B '                 w                    w                    w                  '
 DC.B '                 w                    w                    w                  '
 DC.B '                 w                    w                    w                  '
 DC.B '                 w                    w                    w                  '
 DC.B '                 w                    w                    w                  '

STR_LEVEL9 
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '     ww                             ww                                        '
 DC.B '       ww                             ww                                      '
 DC.B '         ww                             ww                                    '
 DC.B '          ww                              ww                                  '
 DC.B '            ww                              ww                                '
 DC.B '              ww                              ww                              '
 DC.B '                ww                              ww                            '
 DC.B '                  ww                              ww                          '
 DC.B '                    ww                              ww                        '
 DC.B '                      ww                              ww                      '
 DC.B '                        ww                              ww                    '
 DC.B '                          ww                              ww                  '
 DC.B '                            ww                              ww                '
 DC.B '                              ww                              ww              '
 DC.B '                                ww                              ww            '
 DC.B '                                  ww                              ww          '
 DC.B '                                    ww                              ww        '
 DC.B '                                      ww                              ww      '
 DC.B '                                                                              '
 DC.B '                                                                              '

STR_LEVEL10 
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '          wwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwww             '
 DC.B '                                                                              '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '                                                                              '
 DC.B '          wwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwww             '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '          wwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwww             '
 DC.B '                                                                              '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '                                                                              '
 DC.B '          wwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwww             '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '

STR_LEVEL11 
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '        wwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwww           '
 DC.B '                                                                  w           '
 DC.B '                                                                  w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                                     '
 DC.B '        w                                                                     '
 DC.B '        wwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwww           '
 DC.B '                                                                              '
 DC.B '                                                                              '


STR_LEVEL12 
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '        wwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwww           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        wwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwww           '
 DC.B '                                                                              '
 DC.B '                                                                              '


STR_LEVEL13 
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '        wwwwwwwwwwwwwww                                           w           '
 DC.B '        w              ww                                         w           '
 DC.B '        w                ww                                       w           '
 DC.B '        w                  ww                                     w           '
 DC.B '        w                    ww                                   w           '
 DC.B '        w                      ww                                 w           '
 DC.B '        w                        ww                               w           '
 DC.B '        w                          ww                             w           '
 DC.B '        w                            ww                           w           '
 DC.B '        w                              ww                         w           '
 DC.B '        w                                ww                       w           '
 DC.B '        w                                  ww                     w           '
 DC.B '        w                                    ww                   w           '
 DC.B '        w                                      ww                 w           '
 DC.B '        w                                        ww               w           '
 DC.B '        w                                          wwwwwwwwwwwwwwww           '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '


STR_LEVEL14 
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '        wwwwwwwwwwww                                wwwwwwwwwwwwwww           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w          wwwwwww                     wwwwwwww           w           '
 DC.B '        w         w                                    w          w           '
 DC.B '        w         w                                    w          w           '
 DC.B '        w         w                                    w          w           '
 DC.B '        w         w                                    w          w           '
 DC.B '        w         w                                    w          w           '
 DC.B '        w          wwwwwww                     wwwwwwww           w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        wwwwwwwwwwwwww                              wwwwwwwwwwwwwww           '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '


STR_LEVEL15 
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '        wwwwwwwwwwww                                wwwwwwwwwwwwwww           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w         wwwwwwww                     wwwwwwwww          w           '
 DC.B '        w                 w                   w                   w           '
 DC.B '        w                 w                   w                   w           '
 DC.B '        w                 w                   w                   w           '
 DC.B '        w                 w                   w                   w           '
 DC.B '        w                 w                   w                   w           '
 DC.B '        w                 w                   w                   w           '
 DC.B '        w         wwwwwwww                     wwwwwwwww          w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        wwwwwwwwwwwwww                              wwwwwwwwwwwwwww           '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '


STR_LEVEL16
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '        wwwwwwwwwwww         wwwwwwwwwwwwwww          wwwwwwwwwwwww           '
 DC.B '        w           ww                              ww            w           '
 DC.B '        w             ww                          ww              w           '
 DC.B '        w               ww                      ww                w           '
 DC.B '        w                 ww     wwwwwww      ww                  w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                 ww     wwwwwww      ww                  w           '
 DC.B '        w               ww                      ww                w           '
 DC.B '        w             ww                          ww              w           '
 DC.B '        w           ww                              ww            w           '
 DC.B '        wwwwwwwwwwww         wwwwwwwwwwwwwww          wwwwwwwwwwwww           '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '


STR_LEVEL17
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '        wwwwwwwwwwwww        wwwwwwwwwwwwwww         wwwwwwwwwwwwww           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                    wwwwwwwwwwwwwww              wwwwwwwww           '
 DC.B '        wwwwwwwww                                                 w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        wwwwwwwww                                         wwwwwwwww           '
 DC.B '        w                    wwwwwwwwwwwwwww                      w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        w                                                         w           '
 DC.B '        wwwwwwwwwwwww        wwwwwwwwwwwwwww         wwwwwwwwwwwwww           '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '


STR_LEVEL18
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '        wwwwwwwwwwwww        wwwwwwwwwwwwwww         wwwwwwwwwwwwww           '
 DC.B '                     ww                            ww                         '
 DC.B '                       ww                        ww                           '
 DC.B '                         ww                    ww                             '
 DC.B '                           wwwwwwwwwwwwwwwwwwww                               '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                           wwwwwwwwwwwwwwwwwwww                               '
 DC.B '                         ww                    ww                             '
 DC.B '                       ww                        ww                           '
 DC.B '                     ww                            ww                         '
 DC.B '        wwwwwwwwwwwww        wwwwwwwwwwwwwww         wwwwwwwwwwwwww           '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '


STR_LEVEL19
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          wwwwwwwwwwww          wwwwwwwwwwww          wwwwwwwwwwww            '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '          wwwwwwwwwwww          wwwwwwwwwwww          wwwwwwwwwwww            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '                                                                              '
 DC.B '                                                                              '


STR_LEVEL20
 DC.B '                                                                              '
 DC.B '            wwwwwwww              wwwwwwww              wwwwwwww              '
 DC.B '                                                                              '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '                                                                              '
 DC.B '            wwwwwwww              wwwwwwww              wwwwwwww              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '            wwwwwwww              wwwwwwww              wwwwwwww              '
 DC.B '                                                                              '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '                                                                              '
 DC.B '            wwwwwwww              wwwwwwww              wwwwwwww              '
 DC.B '                                                                              '

STR_LEVEL21
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '          wwwwwwwwwwww          wwwwwwwwwwww          wwwwwwwwwwww            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          wwwwwwwwwww           wwwwwwwwwwww          w            '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '                                                                              '
 DC.B '          w          wwwwwwwwwwww          wwwwwwwwwwww          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          w          w          w          w          w          w            '
 DC.B '          wwwwwwwwwwww          wwwwwwwwwwww          wwwwwwwwwwww            '
 DC.B '                                                                              '
 DC.B '                                                                              '

STR_LEVEL22 
 DC.B '                   w                   w                   w                  '
 DC.B '         w                   w                   w                   w        '
 DC.B '                   w                   w                   w                  '
 DC.B '         w                   w                   w                   w        '
 DC.B '                   w                   w                   w                  '
 DC.B '         w                   w                   w                   w        '
 DC.B '                   w                   w                   w                  '
 DC.B '         w                   w                   w                   w        '
 DC.B '                   w                   w                   w                  '
 DC.B '         w                   w                   w                   w        '
 DC.B '                   w                   w                   w                  '
 DC.B '         w                   w                   w                   w        '
 DC.B '                   w                   w                   w                  '
 DC.B '         w                   w                   w                   w        '
 DC.B '                   w                   w                   w                  '
 DC.B '         w                   w                   w                   w        '
 DC.B '                   w                   w                   w                  '
 DC.B '         w                   w                   w                   w        '
 DC.B '                   w                   w                   w                  '
 DC.B '         w                   w                   w                   w        '
 DC.B '                   w                   w                   w                  '
 DC.B '         w                   w                   w                   w        '



        END NIBBLES
