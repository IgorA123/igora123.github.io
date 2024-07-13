---
layout: post
title: Exercício I/O
date: 2024-07-13 12:00:00 -0300
categories: x86 assembly
---

# Ambiente

`Linux 5.10.16.3-microsoft-standard-WSL2 #1 SMP x86_64 x86_64 x86_64 GNU/Linux`

`NASM version 2.15.05`

`gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0`

`gdb (Ubuntu 12.1-0ubuntu1~22.04) 12.1`

# Biblioteca de entrada/saída

| Function       | Definition |
| -------------- | ---------- |
| exit           | Accepts an exit code and terminates current process. |
| string_length  | Accepts a pointer to a string and returns its length.| 
| print_string   | Accepts a pointer to a null-terminated string and prints it to **stdout**. |
| print_char     | Accepts a character code directly as its first argument and prints it to **stdout**. |
| print_newline  | Prints a character with code `0xA`. |
| print_uint     | Outputs an unsigned 8-byte integer in decimal format. We suggest you create a buffer on the stack and store the division results there. Each time you divide the last value by 10 and store the corresponding digit inside the buffer. Do not forget, that you should transform each digit into its ASCII code (e.g., `0x04` becomes `0x34`). |
| print_int      | Output a signed 8-byte integer in decimal format. |
| read_char      | Read one character from **stdin and return it**. If the end of input stream occurs, return 0. |
| read_word      | Accepts a buffer address and size as arguments. Reads next word from **stdin** (skipping whitespaces into buffer). Stops and returns 0 if word is too big for the buffer specified; otherwise returns a buffer address. This function should null-terminate the accepted string. |
| parse_uint     | Accepts a null-terminated string and tries to parse an unsigned number from its start. Returns the number parsed in `rax`, its characters count in `rdx`. |
| parse_int      | Accepts a null-terminated string and tries to parse a signed number from its start. Returns the number parsed in `rax`; its characters count in `rdx` (including sign if any). No spaces between sign and digits are allowed. |
| string_equals  | Accepts two pointers to strings and compares them. Returns 1 if they are equal, otherwise 0. |
| string_copy    | Accepts a pointer to a string, a pointer to a buffer, and buffer’s length. Copies string to the destination. The destination address is returned if the string fits the buffer; otherwise zero s returned. |



# Minha implementação


```nasm
; por ser iniciante, algumas implementações podem estar ineficientes e/ou mal otimizadas, use por sua conta e risco

section .text

exit:
    mov rax, 60
    xor rdi, rdi
    syscall

string_length:
    xor rax, rax
    .loop:
    cmp byte [rdi+rax], 0
    je .end
    inc rax
    jmp .loop
    .end:
    ret

print_string:
    push rdi                ; inserir ponteiro/string na pilha

    call string_length
    pop rsi                 ; resgatar em outro registrador

    mov rdx, rax            ; comprimento
    mov rax, 1              ; write
    mov rdi, 1              ; stdout
    syscall
    ret

print_char:
    push rdi
    mov rdi, rsp

    call print_string
    pop rdi
    ret

print_newline:
    mov rdi, 10
    jmp print_char

print_uint:
    mov rax, rdi            ; rdi contém a string, mova para rax
    mov rdi, rsp            ; rsp contém o endereço do topo da pilha, mova para rdi
    push 0                  ; terminador nulo (8 bytes)
    sub rsp, 16             ; alocar espaço na pilha (buffer)

    dec rdi                 ; decrementar rdi para apontar para o final do buffer (próxima posição livre)
    mov r8, 10              ; valor a ser usado nas divisões
    .div_loop:
    xor rdx, rdx            ; zerar rdx para armazenar os restos das divisões
    div r8                  ; divisão: rax fica com o quociente, resto em rdx
    add dl, 0x30            ; adicionar 0x30 na menor parte de rdx (dl) 
    dec rdi                 ; ir para o próximo espaço livre do buffer
    mov [rdi], dl           ; mover o caractere ascii no local apontado

    test rax, rax
    jnz .div_loop

    call print_string

    add rsp, 24             ; restaurar a pilha
    ret

print_int:
    test rdi, rdi           ; verifica se rdi é igual a 0 para saber se é positivo ou negativo
    jns print_uint          ; se for positivo, vá para print_uint
    push rdi                ; guarde rdi na pilha
    mov rdi, '-'            ; mover o sinal de negativo para rdi
    call print_char         ; chamar print_char para imprimir '-'
    pop rdi                 ; recuperar rdi da pilha
    neg rdi                 ; negativar rdi (torná-lo positivo nesse caso)
    call print_uint         ; chamar print_uint

read_char:
    push 0                  ; empilhar um valor inicial 
    mov rsi, rsp            ; apontar o buffer para o rsi
    mov rdx, 1              ; quantidade de bytes a serem lidos
    mov rdi, 0              ; stdin  
    mov rax, 0              ; read()
    syscall
    pop rax                 ; guardar o retorno em rax
    ret 

read_word:
    push r14                ; salvar o estado dos regs r14 e r15
    push r15                ; r14 será o contador de caracteres e r15 armazenará o endereço do buffer

    xor r14, r14
    mov r15, rsi 
    dec r15                 ; técnica para comparar durante a leitura dos caracteres

    .check_first_chars:
    push rdi
    call read_char
    pop rdi

    cmp al, ' '             ; espaço em branco
    je .check_first_chars

    cmp al, 10              ; newline \n
    je .check_first_chars

    cmp al, 13              ; carriage return \r
    je .check_first_chars

    cmp al, 9               ; tab \t
    je .check_first_chars

    test al, al
    jz .end

    .readloop:
    mov byte [rdi + r14], al ; guardar o caractere
    inc r14 
    
    push rdi
    call read_char
    pop rdi   

    cmp al, ' '             ; espaço em branco
    je .end

    cmp al, 10              ; newline \n
    je .end

    cmp al, 13              ; carriage return \r
    je .end

    cmp al, 9               ; tab \t
    je .end

    test al, al 
    jz .end

    cmp r14, r15
    je .max 

    jmp .readloop

    .end:
    mov byte [rdi + r14], 0 ; caractere nulo 
    mov rax, rdi            ; preparar o 'ret' 
    mov rdx, r14            ; quantidade lida

    pop r14
    pop r15
    ret

    .max:
    xor rax, rax            ; devolver '0' se o máximo do buffer for atingido
    pop r14
    pop r15
    ret

; rdi points to a string
; returns rax: number, rdx : length
parse_uint:
    mov r8, 10              ; r8 será usado para multiplicações
    xor rax, rax
    xor rdx, rdx
    .parseloop:
    cmp byte [rdi], 0       ; ver se a string está no final
    jz .end

    cmp byte [rdi], '0'     ; ver se o caractere está abaixo de 0x30 ASCII
    jl .end
    cmp byte [rdi], '9'     ; ver se o caractere está acima de 0x39 ASCII
    ja .end

    movzx rcx, byte [rdi]   ; se chegou aqui, é um dígito, copie para rcx no tamanho correto (zero-extended)

    sub rcx, '0'            ; subtraia 0x30 para "converter" o ASCII
    push rdx
    mul r8                  ; multiplicar o valor de rax por 10 (e liberar um dígito à direita)
    pop rdx

    add rax, rcx            ; "cole" o valor convertido no espaço liberado

    inc rdx
    inc rdi

    jmp .parseloop

    .end:
    ret

; rdi points to a string
; returns rax: number, rdx : length
parse_int:
    cmp byte [rdi], '-'     ; comparar se o caractere atual é um sinal de negativo
    je .signed
    jmp parse_uint

    .signed:
    inc rdi                 ; ir para o próximo caractere
    call parse_uint
    neg rax                 ; inverta o sinal do número retornado pelo parse_uint

    test rdx, rdx           ; verificar se o comprimento do número é 0
    jz .error

    inc rdx                 ; se não for 0, adicione 1
    ret

    .error:
    xor rax, rax
    ret

string_equals:
    xor rax, rax
    mov r8b, byte [rdi]     ; mover o primeiro byte da string de origem para r8
    mov r9b, byte [rsi]     ; mover o primeiro byte da string de destino para r9
    cmp r8b,  r9b           ; comparar se são iguais
    jne .not_equal          ; se não são iguais, retorna 0
    cmp r8b, 0              ; comparar se a string de origem está no fim
    je .equal               ; se são iguais, retorna 1
    inc rdi                 ; ir para o próximo byte 
    inc rsi                 ; ir para o próximo byte 
    jmp string_equals

    .equal:
    mov rax, 1
    ret

    .not_equal:
    xor rax, rax
    ret

string_copy:
    xor rcx, rcx            ; zerar o comparador de comprimento
    cmp rdx, 0              ; ver se o comprimento é 0 ou negativo
    jle .not_good

    .copy_loop:
    cmp byte [rdi], 0       ; ver se a string de origem está no fim
    jz .end 
    mov al, byte [rdi]      ; copiar byte (no endereço) da string de origem para al
    mov byte [rsi], al      ; copiar byte de al para string de destino (no endereço)
    inc rdi                 ; ir para o próximo endereço
    inc rsi                 ; ir para o próximo endereço
    inc rcx                 ; aumentar comparador de comprimento

    cmp rcx, rdx            ; comparar os comprimentos
    jl .copy_loop           ; se não excedeu, continua o loop

    jmp .not_good           ; se excedeu, retorna 0

    .end:
    mov byte [rsi], 0       ; terminador nulo na string de destino
    mov rax, rsi            ; mover endereço final da string de destino para rax
    sub rax, rcx            ; subtrair a quantidade de bytes copiados para retornar o endereço inicial da string de destino
    ret

    .not_good:
    xor rax, rax
    ret
```

[Solução do autor no Github](https://github.com/Apress/low-level-programming/blob/master/assignments/1_io_library/teacher/lib.inc)

# Arquivo test.py

[Original do Github](https://github.com/Apress/low-level-programming/blob/master/assignments/1_io_library/teacher/test.py)

[Código modificado para otimizações de versão do python](https://github.com/gyu-don/low-level-programming/blob/test_py3/assignments/1_io_library/stud/test.py)


```python
#!/usr/bin/python

from __future__ import print_function, division
try:
    bytes('foo', 'utf-8')
except:
    u8_bytes = lambda a: a
    u8_str = lambda a: a
else:
    u8_bytes = lambda a: bytes(a, 'utf-8')
    u8_str = lambda a: str(a, 'utf-8')

import os
import subprocess
import sys
import re
import sys
from subprocess import CalledProcessError, Popen, PIPE

try:
    from termcolor import colored
except ImportError:
    from termcolor_py2 import colored

#-------helpers---------------

def starts_uint( s ):
    matches = re.findall('^\d+', s)
    if matches:
        return (int(matches[0]), len(matches[0]))
    else:
        return (0, 0)

def starts_int( s ):
    matches = re.findall('^-?\d+', s)
    if matches:
        return (int(matches[0]), len(matches[0]))
    else:
        return (0, 0)

def unsigned_reinterpret(x):
    if x < 0:
        return x + 2**64
    else:
        return x

def first_or_empty( s ):
    sp = s.split()
    if sp == [] : 
        return ''
    else:
        return sp[0]

#-----------------------------

def compile( fname, text ):
    f = open( fname + '.asm', 'w')
    f.write( text )
    f.close()

    if subprocess.call( ['nasm', '-f', 'elf64', fname + '.asm', '-o', fname+'.o'] ) == 0 and subprocess.call( ['ld', '-o' , fname, fname+'.o'] ) == 0:
             print(' ', fname, ': compiled')
             return True
    else: 
        print(' ', fname, ': failed to compile')
        return False


def launch( fname, seed = '' ):
    output = ''
    try:
        p = Popen(['./'+fname], shell=None, stdin=PIPE, stdout=PIPE)
        (output, err) = p.communicate(input=u8_bytes(seed))
        return (u8_str(output), p.returncode)
    except CalledProcessError as exc:
        return (exc.output, exc.returncode)
    else:
        return (u8_str(output), 0)



def test_asm( text, name = 'dummy',  seed = '' ):
    if compile( name, text ):
        r = launch( name, seed )
        #os.remove( name )
        #os.remove( name + '.o' )
        #os.remove( name + '.asm' )
        return r 
    return None 

class Test:
    name = ''
    string = lambda x : x
    checker = lambda input, output, code : False

    def __init__(self, name, stringctor, checker):
        self.checker = checker
        self.string = stringctor
        self.name = name
    def perform(self, arg):
        res = test_asm( self.string(arg), self.name, arg)
        if res is None:
            return False
        (output, code) = res
        print('"', arg,'" ->',  res)
        return self.checker( arg, output, code )

before_call="""
mov rdi, -1
mov rsi, -1
mov rax, -1
mov rcx, -1
mov rdx, -1
mov r8, -1
mov r9, -1
mov r10, -1
mov r11, -1
push rbx
push rbp
push r12 
push r13 
push r14 
push r15 
"""
after_call="""
cmp r15, [rsp] 
jne .convention_error
pop r15
cmp r14, [rsp] 
jne .convention_error
pop r14
cmp r13, [rsp] 
jne .convention_error
pop r13
cmp r12, [rsp] 
jne .convention_error
pop r12
cmp rbp, [rsp] 
jne .convention_error
pop rbp
cmp rbx, [rsp] 
jne .convention_error
pop rbx

jmp continue

.convention_error:
    mov rax, 1
    mov rdi, 2
    mov rsi, err_calling_convention
    mov rdx,  err_calling_convention.end - err_calling_convention
    syscall
    mov rax, 60
    mov rdi, -41
    syscall
section .data
err_calling_convention: db "You did not respect the calling convention! Check that you handled caller-saved and callee-saved registers correctly", 10
.end:
section .text
continue:
"""
tests=[ Test('string_length',
             lambda v : """section .data
        str: db '""" + v + """', 0
        section .text
        %include "lib.inc"
        global _start
        _start:
        """ + before_call + """
        mov rdi, str
        call string_length
        """ + after_call + """
        mov rdi, rax
        mov rax, 60
        syscall""",
        lambda i, o, r: r == len(i)
         ),

        Test('print_string',
             lambda v : """section .data
        str: db '""" + v + """', 0
        section .text
        %include "lib.inc"
        global _start 
        _start:
        """ + before_call + """
        mov rdi, str
        call print_string
        """ + after_call + """

        mov rax, 60
        xor rdi, rdi
        syscall""", 
        lambda i,o,r: i == o),

        Test('print_char',
            lambda v:""" section .text
        %include "lib.inc"
        global _start 
        _start:
        """ + before_call + """
        mov rdi, '""" + v + """'
        call print_char
        """ + after_call + """
        mov rax, 60
        xor rdi, rdi
        syscall""", 
        lambda i,o,r: i == o),

        Test('print_uint',
            lambda v: """section .text
        %include "lib.inc"
        global _start 
        _start:
        """ + before_call + """
        mov rdi, """ + v + """
        call print_uint
        """ + after_call + """
        mov rax, 60
        xor rdi, rdi
        syscall""", 
        lambda i, o, r: o == str(unsigned_reinterpret(int(i)))),
        
        Test('print_int',
            lambda v: """section .text
        %include "lib.inc"
        global _start 
        _start:
        """ + before_call + """
        mov rdi, """ + v + """
        call print_int
        """ + after_call + """
        mov rax, 60
        xor rdi, rdi
        syscall""", 
        lambda i, o, r: o == i),

        Test('read_char',
             lambda v:"""section .text
        %include "lib.inc"
        global _start 
        _start:
        """ + before_call + """
        call read_char
        """ + after_call + """
        mov rdi, rax
        mov rax, 60
        syscall""", 
        lambda i, o, r: (i == "" and r == 0 ) or ord( i[0] ) == r ),

        Test('read_word',
             lambda v:"""
        section .data
        word_buf: times 20 db 0xca
        section .text
        %include "lib.inc"
        global _start 
        _start:
        """ + before_call + """
        mov rdi, word_buf
        mov rsi, 20 
        call read_word
        """ + after_call + """
        mov rdi, rax
        call print_string

        mov rax, 60
        xor rdi, rdi
        syscall""", 
        lambda i, o, r: i == o),

        Test('read_word_length',
             lambda v:"""
        section .data
        word_buf: times 20 db 0xca
        section .text
        %include "lib.inc"
        global _start 
        _start:
        """ + before_call + """
        mov rdi, word_buf
        mov rsi, 20 
        call read_word
        """ + after_call + """
        
        mov rax, 60
        mov rdi, rdx
        syscall""",
        lambda i, o, r: len(first_or_empty(i.strip())) == r or (len(first_or_empty(i)) == 0 and r == 0)),
        #lambda i, o, r: len(i) == r or len(i) > 19),

        Test('read_word_too_long',
             lambda v:"""
        section .data
        word_buf: times 20 db 0xca
        section .text
        %include "lib.inc"
        global _start 
        _start:
        """ + before_call + """
        mov rdi, word_buf
        mov rsi, 20 
        call read_word
        """ + after_call + """

        mov rdi, rax
        mov rax, 60
        syscall""", 
        lambda i, o, r: ( (not len(i) > 19) and r != 0 ) or  r == 0 ),

        Test('parse_uint',
             lambda v: """section .data
        input: db '""" + v  + """', 0
        section .text
        %include "lib.inc"
        global _start 
        _start:
        """ + before_call + """
        mov rdi, input
        call parse_uint
        """ + after_call + """
        push rdx
        mov rdi, rax
        call print_uint
        mov rax, 60
        pop rdi
        syscall""", 
        lambda i,o,r:  starts_uint(i)[0] == int(o) and r == starts_uint( i )[1]),
        
        Test('parse_int',
             lambda v: """section .data
        input: db '""" + v  + """', 0
        section .text
        %include "lib.inc"
        global _start 
        _start:
        """ + before_call + """
        mov rdi, input
        call parse_int
        """ + after_call + """
        push rdx
        mov rdi, rax
        call print_int
        pop rdi
        mov rax, 60
        syscall""", 
        lambda i,o,r: (starts_int( i )[1] == 0 and int(o) == 0) or (starts_int(i)[0] == int(o) and r == starts_int( i )[1] )),

        Test('string_equals',
             lambda v: """section .data
             str1: db '""" + v + """',0
             str2: db '""" + v + """',0
        section .text
        %include "lib.inc"
        global _start
        _start:
        """ + before_call + """
        mov rdi, str1
        mov rsi, str2
        call string_equals
        """ + after_call + """
        mov rdi, rax
        mov rax, 60
        syscall""",
        lambda i,o,r: r == 1),
 
        Test('string_equals not equals',
             lambda v: """section .data
             str1: db '""" + v + """',0
             str2: db '""" + v + """!!',0
        section .text
        %include "lib.inc"
        global _start
        _start:
        """ + before_call + """
        mov rdi, str1
        mov rsi, str2
        call string_equals
        """ + after_call + """
        mov rdi, rax
        mov rax, 60
        syscall""",
        lambda i,o,r: r == 0),

        Test('string_copy',
            lambda v: """
        section .data
        arg1: db '""" + v + """', 0
        arg2: times """ + str(len(v) + 1) +  """ db  66
        section .text
        %include "lib.inc"
        global _start 
        _start:
        """ + before_call + """
        mov rdi, arg1
        mov rsi, arg2
        mov rdx, """ + str(len(v) + 1) + """
        call string_copy
        """ + after_call + """
        mov rdi, arg2 
        call print_string
        mov rax, 60
        xor rdi, rdi
        syscall""", 
        lambda i,o,r: i == o),

        Test('string_copy_too_long',
            lambda v: """
            section .rodata
            err_too_long_msg: db "string is too long", 10, 0
            section .data
        arg1: db '""" + v + """', 0
        arg2: times """ + str(len(v)//2)  +  """ db  66
        section .text
        %include "lib.inc"
        global _start 
        _start:
        """ + before_call + """
        mov rdi, arg1
        mov rsi, arg2
        mov rdx, """ + str(len(v)//2) + """
        call string_copy
        test rax, rax
        jnz .good
        mov rdi, err_too_long_msg 
        call print_string
        jmp _exit
        .good:
        """ + after_call + """
        mov rdi, arg2 
        call print_string
        _exit:
        mov rax, 60
        xor rdi, rdi
        syscall""", 
        lambda i,o,r: o.find("too long") != -1 ) 
        ]


inputs= {'string_length' 
        : [ 'asdkbasdka', 'qwe qweqe qe', ''],
         'print_string'  
         : ['ashdb asdhabs dahb', ' ', ''],
         'print_char'    
         : "a c",
         'print_uint'    
         : ['-1', '12345234121', '0', '12312312', '123123'],
         'print_int'     
         : ['-1', '-12345234121', '0', '123412312', '123123'],
         'read_char'            
         : ['-1', '-1234asdasd5234121', '', '   ', '\t   ', 'hey ya ye ya', 'hello world' ],
         'read_word'            
         : ['-1'], # , '-1234asdasd5234121', '', '   ', '\t   ', 'hey ya ye ya', 'hello world' ],
         'read_word_length'     
         : ['-1', '-1234asdasd5234121', '', '   ', '\t   ', 'hey ya ye ya', 'hello world' ],
         'read_word_too_long'     
         : [ 'asdbaskdbaksvbaskvhbashvbasdasdads wewe', 'short' ],
         'parse_uint'           
         : ["0", "1234567890987654321hehehey", "1" ],
         'parse_int'                
         : ["0", "1234567890987654321hehehey", "-1dasda", "-eedea", "-123123123", "1" ],
         'string_equals'            
         : ['ashdb asdhabs dahb', ' ', '', "asd" ],
         'string_equals not equals' 
         : ['ashdb asdhabs dahb', ' ', '', "asd" ],
         'string_copy'   
         : ['ashdb asdhabs dahb', ' ', ''],
         'string_copy_too_long'   
         : ['ashdb asdhabs dahb', ' ', ''],
}
              
if __name__ == "__main__":
    found_error = False
    for t in tests:
        for arg in inputs[t.name]:
            if not found_error:
                try:
                    print('          testing', t.name,'on "'+ arg +'"')
                    res = t.perform(arg)
                    if res: 
                        print('  [', colored('  ok  ', 'green'), ']')
                    else:
                        print('* [ ', colored('fail', 'red'),  ']')
                        found_error = True
                except KeyError:
                    print('* [ ', colored('fail', 'red'),  '] with exception' , sys.exc_info()[0])
                    found_error = True
    if found_error:
        print('Not all tests have been passed')
    else:
        print(colored( "Good work, all tests are passed", 'green'))
```

## Output gerado da minha implementação:


```bash
          testing string_length on "asdkbasdka"
  string_length : compiled
" asdkbasdka " -> ('', 10)
  [   ok   ]
          testing string_length on "qwe qweqe qe"
  string_length : compiled
" qwe qweqe qe " -> ('', 12)
  [   ok   ]
          testing string_length on ""
  string_length : compiled
"  " -> ('', 0)
  [   ok   ]
          testing print_string on "ashdb asdhabs dahb"
  print_string : compiled
" ashdb asdhabs dahb " -> ('ashdb asdhabs dahb', 0)
  [   ok   ]
          testing print_string on " "
  print_string : compiled
"   " -> (' ', 0)
  [   ok   ]
          testing print_string on ""
  print_string : compiled
"  " -> ('', 0)
  [   ok   ]
          testing print_char on "a"
  print_char : compiled
" a " -> ('a', 0)
  [   ok   ]
          testing print_char on " "
  print_char : compiled
"   " -> (' ', 0)
  [   ok   ]
          testing print_char on "c"
  print_char : compiled
" c " -> ('c', 0)
  [   ok   ]
          testing print_uint on "-1"
  print_uint : compiled
" -1 " -> ('18446744073709551615', 0)
  [   ok   ]
          testing print_uint on "12345234121"
  print_uint : compiled
" 12345234121 " -> ('12345234121', 0)
  [   ok   ]
          testing print_uint on "0"
  print_uint : compiled
" 0 " -> ('0', 0)
  [   ok   ]
          testing print_uint on "12312312"
  print_uint : compiled
" 12312312 " -> ('12312312', 0)
  [   ok   ]
          testing print_uint on "123123"
  print_uint : compiled
" 123123 " -> ('123123', 0)
  [   ok   ]
          testing print_int on "-1"
  print_int : compiled
" -1 " -> ('-1', 0)
  [   ok   ]
          testing print_int on "-12345234121"
  print_int : compiled
" -12345234121 " -> ('-12345234121', 0)
  [   ok   ]
          testing print_int on "0"
  print_int : compiled
" 0 " -> ('0', 0)
  [   ok   ]
          testing print_int on "123412312"
  print_int : compiled
" 123412312 " -> ('123412312', 0)
  [   ok   ]
          testing print_int on "123123"
  print_int : compiled
" 123123 " -> ('123123', 0)
  [   ok   ]
          testing read_char on "-1"
  read_char : compiled
" -1 " -> ('', 45)
  [   ok   ]
          testing read_char on "-1234asdasd5234121"
  read_char : compiled
" -1234asdasd5234121 " -> ('', 45)
  [   ok   ]
          testing read_char on ""
  read_char : compiled
"  " -> ('', 0)
  [   ok   ]
          testing read_char on "   "
  read_char : compiled
"     " -> ('', 32)
  [   ok   ]
          testing read_char on "	   "
  read_char : compiled
" 	    " -> ('', 9)
  [   ok   ]
          testing read_char on "hey ya ye ya"
  read_char : compiled
" hey ya ye ya " -> ('', 104)
  [   ok   ]
          testing read_char on "hello world"
  read_char : compiled
" hello world " -> ('', 104)
  [   ok   ]
          testing read_word on "-1"
  read_word : compiled
" -1 " -> ('-1', 0)
  [   ok   ]
          testing read_word_length on "-1"
  read_word_length : compiled
" -1 " -> ('', 2)
  [   ok   ]
          testing read_word_length on "-1234asdasd5234121"
  read_word_length : compiled
" -1234asdasd5234121 " -> ('', 18)
  [   ok   ]
          testing read_word_length on ""
  read_word_length : compiled
"  " -> ('', 0)
  [   ok   ]
          testing read_word_length on "   "
  read_word_length : compiled
"     " -> ('', 0)
  [   ok   ]
          testing read_word_length on "	   "
  read_word_length : compiled
" 	    " -> ('', 0)
  [   ok   ]
          testing read_word_length on "hey ya ye ya"
  read_word_length : compiled
" hey ya ye ya " -> ('', 3)
  [   ok   ]
          testing read_word_length on "hello world"
  read_word_length : compiled
" hello world " -> ('', 5)
  [   ok   ]
          testing read_word_too_long on "asdbaskdbaksvbaskvhbashvbasdasdads wewe"
  read_word_too_long : compiled
" asdbaskdbaksvbaskvhbashvbasdasdads wewe " -> ('', 0)
  [   ok   ]
          testing read_word_too_long on "short"
  read_word_too_long : compiled
" short " -> ('', 0)
  [   ok   ]
          testing parse_uint on "0"
  parse_uint : compiled
" 0 " -> ('0', 1)
  [   ok   ]
          testing parse_uint on "1234567890987654321hehehey"
  parse_uint : compiled
" 1234567890987654321hehehey " -> ('1234567890987654321', 19)
  [   ok   ]
          testing parse_uint on "1"
  parse_uint : compiled
" 1 " -> ('1', 1)
  [   ok   ]
          testing parse_int on "0"
  parse_int : compiled
" 0 " -> ('0', 1)
  [   ok   ]
          testing parse_int on "1234567890987654321hehehey"
  parse_int : compiled
" 1234567890987654321hehehey " -> ('1234567890987654321', 19)
  [   ok   ]
          testing parse_int on "-1dasda"
  parse_int : compiled
" -1dasda " -> ('-1', 2)
  [   ok   ]
          testing parse_int on "-eedea"
  parse_int : compiled
" -eedea " -> ('0', 0)
  [   ok   ]
          testing parse_int on "-123123123"
  parse_int : compiled
" -123123123 " -> ('-123123123', 10)
  [   ok   ]
          testing parse_int on "1"
  parse_int : compiled
" 1 " -> ('1', 1)
  [   ok   ]
          testing string_equals on "ashdb asdhabs dahb"
  string_equals : compiled
" ashdb asdhabs dahb " -> ('', 1)
  [   ok   ]
          testing string_equals on " "
  string_equals : compiled
"   " -> ('', 1)
  [   ok   ]
          testing string_equals on ""
  string_equals : compiled
"  " -> ('', 1)
  [   ok   ]
          testing string_equals on "asd"
  string_equals : compiled
" asd " -> ('', 1)
  [   ok   ]
          testing string_equals not equals on "ashdb asdhabs dahb"
  string_equals not equals : compiled
" ashdb asdhabs dahb " -> ('', 0)
  [   ok   ]
          testing string_equals not equals on " "
  string_equals not equals : compiled
"   " -> ('', 0)
  [   ok   ]
          testing string_equals not equals on ""
  string_equals not equals : compiled
"  " -> ('', 0)
  [   ok   ]
          testing string_equals not equals on "asd"
  string_equals not equals : compiled
" asd " -> ('', 0)
  [   ok   ]
          testing string_copy on "ashdb asdhabs dahb"
  string_copy : compiled
" ashdb asdhabs dahb " -> ('ashdb asdhabs dahb', 0)
  [   ok   ]
          testing string_copy on " "
  string_copy : compiled
"   " -> (' ', 0)
  [   ok   ]
          testing string_copy on ""
  string_copy : compiled
"  " -> ('', 0)
  [   ok   ]
          testing string_copy_too_long on "ashdb asdhabs dahb"
  string_copy_too_long : compiled
" ashdb asdhabs dahb " -> ('string is too long\n', 0)
  [   ok   ]
          testing string_copy_too_long on " "
  string_copy_too_long : compiled
"   " -> ('string is too long\n', 0)
  [   ok   ]
          testing string_copy_too_long on ""
  string_copy_too_long : compiled
"  " -> ('string is too long\n', 0)
  [   ok   ]
Good work, all tests are passed
```
