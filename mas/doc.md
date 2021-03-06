## Мнемоники

Мнемоники представляют собой более удобный формат записи последовательности управляющих битов. Они могут принимать до 3
операндов (в зависимости от конкретной мнемоники).

## Метки

Метки используются вместо числовых адресов для переходов по микрокоду, т.е. являются именованными адресами. Любая
метка должна указывать на микроинструкцию, адрес которой будет использован вместо метки. Метки могут быть использованы
с мнемониками `jmp` и `jcnd`. Объявление состоит из имени (последовательность цифр, букв, символов `_`), двоеточия,
следующего сразу после него. На следующей непустой строке должна обязательно находиться микроинструкция, на которую
указывает метка. Если её нет, будет выдана ошибка.

Пример:

```
label1:
    mov a, b
    
label2:

    jmp label2
```

Адрес метке выдается во время распределения во втором проходе. Также адрес может быть установлен заранее, в таком случае
он сохраняется при распределении (подробнее см. "Второй проход" и "@instr").

## Первый проход

Процесс сборки микрокода разделён на первый и второй проход. Во время первого происходит обработка всех определений
меток, обработка управляющих битов микроинструкций и их сохранение для второго прохода, а также включения фрагментов
исходного кода их других файлов директивой `@include`.

## Второй проход

Во время этого прохода меткам выдаются адреса путём их распределения по микрокоду. На один адрес будет назначена одна
метка. Приоритет выдачи адреса по категориям меток (по убыванию):

1. Метки с конкретно указанным вручную адресом сохраняют его. Если вручную на один адрес назначено больше одной метки,
   выдаётся ошибка.
2. Метки, для которых указана главная или префиксная страница, размещаются на них. Если места на странице для всех меток
   не хватит, то будет выдана ошибка.
3. Остальные метки будут распределены по возрастанию по остальным частям микрокода с первого свободного адреса. Если в
   микрокоде нет свободных адресов будет выдана ошибка.

Структура микрокод:

* главная страница `[0x000-0x0ff]` - 256 микроинструкций,
* префиксная страницы `[0x800-0x8ff]` - 256 микроинструкций,
* остальное пространство `[0x100-0x7ff] & [0x900-0xfff]` - 3584 микроинструкций,
* всего микрокода `[0x000-0xfff]` - 4096 микроинструкций.

Эти ограничения обусловлены техническими особенностями микроархитектуры процессора, а не микроассемблером.

## Директивы

Микро-ассемблер поддерживает директивы, которые выполняются во время первого прохода. В коде отмечаются символом `@` и
могут иметь параметр. Занимают одну строку, на которой, за исключением комментариев, не может содержаться других
лексических конструкций.

### @instr

Директива `@instr` служит для указания страницы в микрокоде, на которую будет распределена метка, следующая за ней. Если
метки после директивы не обнаружено, то выдаётся ошибка. Параметр директивы может:

* отсутствовать, в таком случае метка будет распределена в пределах главной странице микрокода;
* содержать специальное значение `pref`, тогда метка распределена на префиксную страницу микрокода;
* содержать конкретный числовой адрес, который будет выдан метке (при распределении будет учтён в первую очередь).

Примеры:

```
@instr          # Метка на главной странице.
label1:

@instr(pref)    # Метка на префиксной странице.
label2:

@instr(f1)      # Метка с адресом 0xF1, соотвествует главной странице.
label3:

@instr(0x80f)   # Метка с адресом 0x80F, соотвествует префиксной странице.
label4:

@instr(0x20A)   # Метка с адресом 0x20A, который не принадлежит ни главной, ни префиксной странице.
label5:
```

Более подробно о распределении см. в разделе "Второй проход".

### @include

Директива `@include` используется для включения в текущий разбираемый файл меток и микроинструкций из другого файла,
который также должен содержать правильную программу на языке микроассемблера. Его путь должен быть указан в качестве
параметра, причём он не должен содержать символа `)`. Все метки и микроинструкции добавляются в текущий разбираемый
файл.

Пример:

```
# ==== Файл '1.mas' ====
@include(2.mas) # Метки label3, label2, а также все микроинструкции будут импортированы. 

label1:
    mov a, b; jmp label3  # Из '2.mas'.
    
# Ошибка, метка label2 уже объявлена в '2.mas'.
label2:
    %nxt


# ==== Файл '2.mas' ====
label3:
    mov b, c
    
label2:
    jmp $0
```

