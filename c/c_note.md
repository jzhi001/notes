# C Notes

## GCC

### Derectives

* `-E` See file after preprocessing

## Macro

### Stringizing

Syntax: #MACRO_ARG
Semantic: convert macro argument to a string

Example:
```C
define JUDGE_THEN_PUTS(cond) do{ \
	if( cond ){ \
		printf("%s is true\n", #cond); \
	} \
} while(0)
```

### Concatenation

Syntax: TEXT ## MACRO_ARG ## TEXT
Semantic: concat macro argument with text

Example:

```C
#define COMMAND(NAME)  { #NAME, NAME ## _command }
```

## Pitfall & trick

### long VS long long

See [Wiki](https://en.wikipedia.org/wiki/C_data_types), `long` type is at least 32 bits and `long long` is at least 64 bits.  

Note: `int` is at least **16** bits which is same as `short`.

### Flexible Array Member

See [this](https://en.wikipedia.org/wiki/Flexible_array_member), I've seen lots of it in Redis source code.

### sizeof Operator

`sizeof` only returns size at compiled time. So if you use flexible array member, you cannot 

use `sizeof`. 

See [this](https://stackoverflow.com/questions/9478044/realloc-is-not-resizing-array-of-pointers)

### Cancel allignment

By default, compiler adds extra bytes for alignment.

See [paper](https://wr.informatik.uni-hamburg.de/_media/teaching/wintersemester_2013_2014/epc-14-haase-svenhendrik-alignmentinc-paper.pdf) or [Wiki](https://en.wikipedia.org/wiki/Data_structure_alignment).

Use "packed" attribute to avoid alignment:

```C
struct __attribute__((__packed__)) intset 
{
    uint8_t eleBytes; //size of one element
    uint64_t size;   //element count
    char elements[];  
} ;

typedef struct __attribute__((__packed__)){ 
    uint8_t eleBytes; //size of one element
    uint64_t size;   //element count
    char elements[]; 
} IntSet;
```

