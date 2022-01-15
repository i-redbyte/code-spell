#Заклинание кода: Гимель
Если взглянуть на типичную программу на ассемблере, то можно заметить, что некоторых инструкций там больше, чем других. Одна из таких инструкций MOV. Эта инструкция отвечает за копирование информации - например, значения ячейки памяти в регистр.

К примеру, мы хотим поместить в EAX число 1. Сделать это просто, нужный опкод в данном случае 0B8h. А как задаётся число, которое надо поместить в регистр? Над этой проблемой билось не одно поколение заклинателей кода, но затем один из них, расшифровав письмена в Книге Двойных Слов, рассказал, что нужное число просто помещается сразу после опкода. То есть заклинание выглядит следующим образом:
```assembly
        db      0B8h    ; mov eax,1
        dd      1
```

Обратите внимание: единица вставлена с помощью директивы dd, а в результате это будет последовательность байт 1,0,0,0. Также обратите внимание, что в последовательности байт единица идёт вначале. Это потому, что в архитектуре процессоров семейства x86 байты слов и двойных слов располагаются от младшего к старшему, а единица, безусловно, находится в младшем байте. Таким образом, я могу переписать вышеприведённое заклинание так:

```assembly
        db      0B8h    ; mov eax, 1
        db      1,0,0,0
```

Оба заклинания делают одно и то же, но я привел их оба, чтобы вы лучше представляли себе, как они устроены на уровне байт. А вместо единицы можно подставить любое 32-битное число.

А что делать, если нужно поместить какое-либо число не в EAX, а в EDI? Данный опкод повинуется правилу, о котором было рассказано в предыдущей главе: к базовому опкоду (0B8h) следует прибавить номер регистра (номер EDI - 7). Это даёт 0B8h+7=0BFh. Давайте проверим это на примере простого заклинания:

```assembly
format PE console
entry start
 
include '..\..\include\kernel.inc'
include '..\..\include\user.inc'
include '..\..\include\macro\stdcall.inc'
include '..\..\include\macro\import.inc'
 
section '.data' data writeable readable
 
_d      db '%d',0
 
section '.code' code executable readable
 
start:
 
        db      0BFh    ; mov edi,100
        dd      100
 
        push    edi
        push    _d
        call    [printf]
        add     esp,8
 
        invoke  ExitProcess,0
 
section '.idata' import data readable writeable
 
                library kernel32,'kernel32.dll',\
                        msvcrt,'msvcrt.dll'
```

Компиляция и последующий запуск программы должны показать, что заклинание работает.

Конечно, заклинатель кода, овладевший копированием числа в регистры общего назначения, может захотеть большего: копирования содержимого регистра в регистр или, чего гляди, регистра в память. Это, разумеется, возможно, но прежде необходимо усвоить несколько вещей.

В языке ассемблера для копирования непосредственного значения (оно же константа, оно же литерал) в регистр, как и содержимого регистра в другой регистр или в память, применяется одна мнемоника - MOV. В тоже время эти варианты имеют разные опкоды. К этому моменту вы уже должны были это подозревать, но теперь, когда я прямо сказал об этом, возможно, волосы на вашей голове (и прочих частях тела) встали дыбом. Или, по меньшей мере, зашевелились. К сожалению (а может и к счастью), это так (не то, что волосы зашевелились, а то, что опкоды разные). Причем зачастую к этим опкодам применяются разные правила.

Начнём с простого случая, с MOV EAX,EDX. Как нам составить заклинание? Базовый опкод инструкции "MOV регистр,регистр" - 08Bh, но правило, которое мы применяли в прошлые разы, здесь не поможет, ведь здесь два операнда, а не один.

Чтобы составить требуемое заклинание, нам придётся обратиться к помощи дополнительного байта, применяемого как раз в таких случаях. Этот байт называется ModR/M, и, если он есть, то располагается сразу после опкода. Сам ModR/M делится на три поля следующим образом:

- Биты 6-7 - Mod
- Биты 3-5 - Reg/Opcode (пока что я буду называть это поле просто Reg)
- Биты 0-2 - R/M

Здесь поля Reg и R/M обычно служат для указания операндов. Помня вышеизложенное, можно легко составить заклинание - в Reg помещаем номер первого регистра, в R/M - номер второго. А в Mod помещаем 3 (почему, пока говорить не буду, ибо вы можете не выдержать обрушившегося на вас знания и цифровые духи будут пировать на ваших костях).

Для удобства ниже даны номера регистров в двоичном формате:
- EAX - 000b
- ECX - 001b
- EDX - 010b
- EBX - 011b
- ESP - 100b
- EBP - 101b
- ESI - 110b
- EDI - 111b

А 3, которое помещается в поле Mod, в двоичной системе счисления равна 11b. Вот и само заклинание для MOV EAX,EDX:
```assembly
        db      08Bh,11000010b
```

Первый байт (08Bh) - это опкод. Второй байт можно логически разделить так: 11(Mod)-000(Reg=EAX)-010(R/M=EDX).

Теперь рассмотрим инструкцию "ADD регистр,регистр". Её базовый опкод равен 03h и она повинуется только что рассмотренному правилу, поэтому мы можем легко составить заклинание, например, для ADD ECX,EBX:
```assembly
        db      03h,11001011b ; 03h(Опкод), 11(Mod)-001(Reg=ECX)-011(R/M=EBX)
```

Теперь давайте составим заклинание для следующей программы:

```assembly
        mov     eax,10
        mov     edx,15
        add     eax,edx
```

В результате в eax должно получиться 25. Вот текст такой программы для тех, кто пока не сумел ввести его сам:
```assembly
format PE console
entry start
 
include '..\..\include\kernel.inc'
include '..\..\include\user.inc'
include '..\..\include\macro\stdcall.inc'
include '..\..\include\macro\import.inc'
 
section '.data' data writeable readable
 
_d      db '%d',0
 
section '.code' code executable readable
 
start:
 
        db      0B8h            ; mov eax, 10
        dd      10
        db      0BAh            ; mov edx, 15
        dd      15
        db      3,11000010b     ; add eax, edx
 
        push    eax
        push    _d
        call    [printf]
        add     esp,8
 
        invoke  ExitProcess,0
 
section '.idata' import data readable writeable
```

Если вы внимательно читали то, о чём говорилось в предыдущей главе, то вспомните, что PUSH EAX можно заменить на 50h. А как заменить PUSH _d? _d - это имя (оно же адрес) переменной, то есть 32-битное число. Опкод "PUSH число" 68h, поэтому соответствующее заклинание такое:

```assembly
        db      68h
        dd      _d
```

ADD ESP,8 заколдовать тоже несложно. Базовый опкод "ADD регистр,число" 81h. Однако для этой инструкции не применимы заклинания, которые мы творили ранее, ведь здесь операндами выступают не два регистра, а только один регистр и число-константа. Как же быть? Очень просто: в Reg записывается 0, а в R/M - номер регистра. В Mod, как и ранее, помещаем 11b, а число должно идти сразу после опкода и байта ModR/M. В итоге получается следующая программа:

```assembly
 
format PE console
entry start
 
include '..\..\include\kernel.inc'
include '..\..\include\user.inc'
include '..\..\include\macro\stdcall.inc'
include '..\..\include\macro\import.inc'
 
section '.data' data writeable readable
 
_d      db '%d',0
 
section '.code' code executable readable
 
start:
 
        db      0B8h            ; mov eax, 10
        dd      10
        db      0BAh            ; mov edx, 15
        dd      15
        db      3,11000010b     ; add eax, edx
 
        db      50h             ; push eax
        db      68h             ; push _d
        dd      _d
 
        call    [printf]
 
        db      81h,11000100b   ; add esp, 8
        dd      8
 
        invoke  ExitProcess,0
 
section '.idata' import data readable writeable
 
                library kernel32,'kernel32.dll',\
                        msvcrt,'msvcrt.dll'
kernel32:       import  ExitProcess,'ExitProcess'
msvcrt:         import  printf,'printf'
```

Надеюсь, вы усвоили материал данной главы. При необходимости проведите курс медитаций или (в крайнем случае) пишите мне по адресу aquila@wasm.ru.

В следующей главе я окончу свой рассказ о байте ModR/M и расскажу о секретах еще одного байта - SIB. 