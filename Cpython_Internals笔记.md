# <p align="center"> Cpython Internals笔记 </p>

#### Lecture 1 - Interpreter and source code overview

1. Interpreter and source code overview.

	```mermaid
	graph LR;
	   srcode[source code]==>compiler(compiler:<br>Not Very Interesting);
	   compiler==>bytecode[bytecode]
	   bytecode==>interpreter(interpreter);
	   interpreter==>output;
	   subgraph Emphasis
	   bytecode
	   interpreter
	   output
	   end
	   style compiler fill:#fff,stroke:#1cc,stroke-width:2px,stroke-dasharray:5,5;
	   style interpreter fill: #f40;
	```

##### Lecture 2 - Opcodes and main interpreter loop

1. 获取bytecode用到的主要命令:

	```python
	# -------------------------------------------
	import dis
	dis.dis(mod)	# 使用dis模块获取mod的bytecode
	# -------------------------------------------
	c = compile(open('test.py').read(), 'test.py', 'exec')	# 返回code object
	# 返回bytecode的数字编码 [100, 0, 0, 90, 0, 0, 100,....]
	[ord(byte) for byte in c.co_code]
	dis.dis(c)	# 返回bytecode
	dir(c) # 返回c中主要信息
	c.co_code # 返回汇编代码
	c.co_consts # 返回常量元组
	```
	```shell
	python -m dis test.py
	```
	*可用byteplay包替代上述命令获得更好的反编译效果<br>
1. 关键代码注释
	1. **Python/ceval.c**
		- ines 693-3021: main interpreter, operate one frame(equivalent one function), returns a python object to whoever calls this function.
		```cpp
		PyObject *PyEval_EvalFrameEx(PyFrameObject *f, int throwflag){
		/* line 689-730: define locale variables to store the locale states
		 * line 698: The pointer to the value stack
		 */
		register PyObject **stack_pointer; /* Next free slot in value stack */
		/* line 825-858: stack manipulation macros
		 * line 919-945: grab everything out from code(frame)
		 * line 964: a giant infinite loop: go through the bytecode one byte a time
		 */
		for (;;) {
			// line 1080: Extract the next opcode
			opcode = NEXTOP();
			// line 1083: Extract args if have arg
			if (HAS_ARG(opcode))
            	oparg = NEXTARG();
			}
			// line 1112: Execute the opcode by switch.
			switch (opcode) {
				/* Py_DECREF/Py_INCREF is used to support the reference count
				   Garbage Collection: like Py_DECREF(v) in POP_TOP.
				*/
				CASE LOAD_FAST: ... break;
				CASE LOAD_CONST: ... break;
				...
				}
			// line 2959-2960: kick out the infinite loop
			if (why != WHY_NOT)
            	break;
			// clean up
			}
		// line 3020: return the result value
		return retval;
		}
		```

##### Lecture 3 - Frames, function calls, and scope

1. 函数调用相关指令预览
	![Pic1](pics/Lecture3_01.png)
	![Pic1](pics/Lecture3_02.png)
1. Definition of code object (PyCodeObject) - **Include/code.h**
    ![Pic1](pics/Lecture3_03.png)
1. Definition of the frame object(PyFrameObject) - **Include/frameobject.h**
	![Pic1](pics/Lecture3_04.png)
1. Terminology Clarity:
	- **Code**: The most primitive thing, a bunch of bytecode. A Code Object has bytecode, and also has some extra semantic information like constants, variable names.
	- **Function**: A function has a code object, and also has a environment  pointer to where was defined.
	- **Frame**: Also has a code object, also has a environment pointer, it's the representation of code at runtime while running it.
	![Pic1](pics/Lecture3_05.png)
1. Detail about the CALL_FUNCTION instruction:
	- line 2671-2686 in **ceval.c**: instruction **CALL_FUNCTION(argc)**，CALL_FUNCTION instruction only need get the argument argc(the low byte of argc indicates the number of positional arguments, high byte number of keyword parameters) from the stack, and interpreter will get proper parameters from stack automatically.

	```cpp
	case CALL_FUNCTION:
        {
            PyObject **sp;
            PCALL(PCALL_ALL);
            sp = stack_pointer;
			#ifdef WITH_TSC
            x = call_function(&sp, oparg, &intr0, &intr1);
			#else
            x = call_function(&sp, oparg);
			#endif
            stack_pointer = sp;
            PUSH(x);
            if (x != NULL)
                continue;
            break;
        }
	```

	- line 3991-4071 in **ceval.c**: **call_function(栈指针, 参数数量oparg)**,只需传递参数数量，具体参数从栈中获取，返回值最终被PUSH(x)语句压入当前栈中。

	```cpp
	static PyObject *call_function(PyObject ***pp_stack, int oparg)
	{
	  // na: number of positional parameters
      int na = oparg & 0xff;
	  // nk: number of keyword parameters
      int nk = (oparg>>8) & 0xff;
      int n = na + 2 * nk;
      PyObject **pfunc = (*pp_stack) - n - 1;
      PyObject *func = *pfunc;
      PyObject *x, *w;

      /* Always dispatch PyCFunction first, because these are
         presumed to be the most frequent callable object.
      */
      if (PyCFunction_Check(func) && nk == 0) {
       ... handle a C function
      } else {
		  /* If it's a regular Python Function, call the fast_function(...) */
          if (PyFunction_Check(func))
              x = fast_function(func, pp_stack, n, na, nk);
          else
	          /* Go this way when call the CALL_FUNCTION instruction
	           * to make instance of a class, at this time 'func' is
		       * a Class Objection instead a true Function Object
	           */
              x = do_call(func, pp_stack, na, nk);
		  Py_DECREF(func);
        }
      /* Clear the stack of the function object.  Also removes
         the arguments in case they weren't consumed already
         (fast_function() and err_args() leave them on the stack).
      */
      while ((*pp_stack) > pfunc) {
          w = EXT_POP(*pp_stack);
          Py_DECREF(w);
          PCALL(PCALL_POP);
      }
      return x;
	}
	```
	- line 4082-4133 in ceval.c: fast_function(函数对象指针, 栈指针, 所有参数在栈中占据空间, 位置参数数量，键参数数量):根据传入的参数从栈中获得参数执行函数对象返回PyObject对象

	```cpp
	static PyObject *
	fast_function(PyObject *func, PyObject ***pp_stack, int n, int na, int nk)
	{
		PyCodeObject *co = (PyCodeObject *)PyFunction_GET_CODE(func);
    	PyObject *globals = PyFunction_GET_GLOBALS(func);
    	PyObject *argdefs = PyFunction_GET_DEFAULTS(func);
		// Create a new frame and assign to f
		PyFrameObject *f;
		f = PyFrame_New(tstate, co, globals, NULL);
		/* Copy arguments from the stack into the new frame
		 * f_localsplus is the storage for localvariables and
		 * value stacks in the new frame(see last line of the definition of PyFrameObject above)
		*/
		fastlocals = f->f_localsplus;
		// stack is the old stack from calling function
        stack = (*pp_stack) - n;
		/* copy arguments from old frame to the new one	(n = na + 2 * nk),
		 * this operation implements the passing of parameters from caller to the calee.
		 */
        for (i = 0; i < n; i++) {
            Py_INCREF(*stack);
            fastlocals[i] = *stack++;
		}
		/* Call the function PyEval_EvalFrameEx(See Lecture2.2.1) that execute
		   the interpreter main loop on the new frame we just created.
		*/
        retval = PyEval_EvalFrameEx(f,0);
        ++tstate->recursion_depth;
        Py_DECREF(f);
        --tstate->recursion_depth;
		/* 返回执行结果，该结果经过上述几个函数层层返回后
		   最终被压入调用了CALL_FUNCTION指令的FRAME的VALUE STACK中
		*/
        return retval;
	}
	```

	- 流程图表述整个CALL_FUNCTION指令执行过程：

	```mermaid
	graph LR;
		CALLFUNCTION(MACRO:CALL_FUNCTION<br>1. 将栈指针和argc传递给call_function函数<br>2. 接收返回值然后压入调用者栈中)--argc-->callfunction[function:call_function<br>根据CALL_FUNCTION指令传递过来的argc参数计算<br>位置参数和关键字参数的数量并从调用者栈中获取<br>Function Object 将Function Object及参数数量等<br>信息作为参数召唤fast_function.];
		callfunction--pp_stack, oparg-->fastfunction[function:fast_function<br>创建Frame对象,将参数从调用者栈中复制<br>到新Frame中,最终将新建的Frame对象传递<br>给执行解释器主循环的PyEval_EvalFrameEx函数.<br>执行完成后将结果retval返回上一级调用者];
		fastfunction--return retval-->callfunction;
		callfunction--return x-->CALLFUNCTION;

	```

##### Lecture 4 - PyObject: The core Python object

1. Overview
	- Inspect an object in Python: ```dir(obj)```-show everyting(Methods, Properties and so on) inside the mode.
	- Everthing in python is an object even a number like 123(instance of PyIntObject), thist is less efficient than C or Java which store numbers directly, but Override an object to add extra features is more flexible.
	- Interface of Python always work with PyObjects, everthing should be wrapped in a python object to be handled in Python. There is implemention of every signle type object(see source code in Object/xxxobject.c).
	- ```id(x)``` return memory location of an object. In python once an object is evaluated, it's memory location won't be changed.
	- ```type(x)``` return type information of an object, every object not only has a id, but also has type information.
	- 查看对象引用数的语句：
	```python
	from sys import getrefcount
	x = ['a', 'b', 'c']
	getrefcount(x)
	```
1. Contents of a generic PyObject:
	![Pic1](pics/Lecture4_01.png)

	```viz
	digraph Python_Object {
		node [shape=record];
		Object[label="{Python Object | {  id |<f0>type | <f1>value | reference-count | ...}}"];
		Type[label="str | list | <f0>int |..."];
		Value[label="a number | <f0>a string | a list | ..."];
		Object:f0 -> Type:f0;
		Object:f1 -> Value:f0;
		PyObject[label="{PyObject | {type pointer | reference-count}}"]
	}
	```
1. Python Objects in the C world(C structual subtype: utilize the same field in C struct to create a type hierarchical in C)
	```viz
	digraph Objects_in_C_world {
		node [shape=record];
		PyObject[label="{PyObject(C struct)|{{Type|Ref-count}|{<f0>pointer|2}}}"]
		PyIntObject[label="{PyIntObject(C struct)|{{Type|Ref-count|Value}|{<f0>pointer|2|30}}}"]
		PyTypeObject[label="{<f1>PyTypeObject(C struct)|{{Type|Ref-count|Value}|{<f0>pointer|5|\"int\"/...}}}"]
		PyTypeObject_type[label="{<f1>PyTypeObject(C struct)|{{Type|Ref-count|Value}|{<f0>pointer|5|\"Type\"}}}"]
		pPyObject[label="*PyObject"];
		PyObject:f0 -> PyTypeObject;
		PyIntObject:f0 -> PyTypeObject;
		PyTypeObject:f0 -> PyTypeObject_type:f1
		PyTypeObject_type:f0 -> PyTypeObject_type:f1
		pPyObject -> {PyObject PyIntObject PyTypeObject PyTypeObject_type}
	}
	```
1. 代码解析
	- line 77-81 and line 101-108 and line 114-116 in **Include/object.h**
	```c
	/* PyObject_HEAD defines the initial segment of every PyObject.
	 * _PyObject_HEAD_EXTRA is always empty
	 */
	#define PyObject_HEAD                   \
   		_PyObject_HEAD_EXTRA                \
   		Py_ssize_t ob_refcnt;               \
   		struct _typeobject *ob_type;
	/* Nothing is actually declared to be a PyObject, but every pointer to
	 	 * a Python object can be cast to a PyObject*.  This is inheritance built
		 * by hand.  Similarly every pointer to a variable-size Python object can,
		 * in addition, be cast to PyVarObject*.
		 */
	typedef struct _object {
   		PyObject_HEAD	// subtype
	} PyObject;

	/* Macros to get reference count, type and size from Python Objects */
	#define Py_REFCNT(ob)           (((PyObject*)(ob))->ob_refcnt)
	#define Py_TYPE(ob)             (((PyObject*)(ob))->ob_type)
	#define Py_SIZE(ob)             (((PyVarObject*)(ob))->ob_size)

	```
	- line 25-26 in **Include/intobject.h**
	```c
	typedef struct {
	    PyObject_HEAD	// Type & Reference Count
	    long ob_ival;	// Value
	} PyIntObject;
	```
	- **Objects/object.c**: generic thing can be done to an object
	```c
	// line 240-248: How was an object created
	PyObject *
	_PyObject_New(PyTypeObject *tp)
	{
	    PyObject *op;
		/* Allocate new memory and put it to a PyObject pointer 'op' */
	    op = (PyObject *) PyObject_MALLOC(_PyObject_SIZE(tp));
	    if (op == NULL)
	        return PyErr_NoMemory();
		/* Call the constructor to initialize the object */
	    return PyObject_INIT(op, tp);
	}
	// line 448-467 and line 406-446: How was an object converted to a string
	PyObject *
	PyObject_Str(PyObject *v)
	{
	    PyObject *res = _PyObject_Str(v);
	    if (res == NULL)
	        return NULL;
	    assert(PyString_Check(res));
	    return res;
	}

	PyObject *
	_PyObject_Str(PyObject *v)
	{
	    PyObject *res;
	    int type_ok;
	    if (v == NULL)
	        return PyString_FromString("<NULL>");
	    if (PyString_CheckExact(v)) {
	        Py_INCREF(v);
	        return v;
	    }
	    if (Py_TYPE(v)->tp_str == NULL)
	        return PyObject_Repr(v);

	    /* It is possible for a type to have a tp_str representation that loops
	       infinitely. */
	    if (Py_EnterRecursiveCall(" while getting the str of an object"))
	        return NULL;
		/* Every type that can be converted to String should have
		   implemented the tp_str method. This is where the
		   dynamic(One Interface call tp_str, and each type
		   implements it's own version of tp_str) comes in.
		   This is the thing that jump into the specific code
		   for each type.
		 */
	    res = (*Py_TYPE(v)->tp_str)(v);
	    if (res == NULL)
	        return NULL;
	    type_ok = PyString_Check(res);
	    if (!type_ok) {
			...
	        return NULL;
	    }
	    return res;
	}

	```

##### Lecture 5 - Example Python data types

1. Objective of this lecture: To understand how Python data types are "subtypes" of the core PyObject.
	- Intro to Python sequence types -- tuples, lists, strings
	- Abstract object interface: **Objects/abstract.c**
	- String type: **Objects/stringobject.c**
1. Differences between Tuple List and String in Python:
	- ```Tuple``` is immutable, ```List``` is mutable and variable length, ```String``` is a sequence of characters(can be accessed by index), it's immutable.
1. About the String object:
	- ![Pic1](pics/Lecture5_01.png)
	- line 35-49 in **Include/stringobject.h**: definition of PyStringObject
	```c
	typedef struct {
	    PyObject_VAR_HEAD	// ref-count & type pointer & size
	    long ob_shash;	// Hash cache
	    int ob_sstate;	// String interned or not
		/* Memmory for storing characters will be allocated by malloc() right after ob_sval
		   Once an PyObject struct is casted to a PyStringObject, we can access
		   characters by offeset to the ob_sval.
		 */
	    char ob_sval[1];

	    /* Invariants(不变的):
	     *     ob_sval contains space for 'ob_size+1' elements.
	     *     ob_sval[ob_size] == 0.
	     *     ob_shash is the hash of the string or -1 if not computed yet.
	     *     ob_sstate != 0 iff the string object is in stringobject.c's
	     *       'interned' dictionary; in this case the two references
	     *       from 'interned' to this object are *not counted* in ob_refcnt.
	     */
	} PyStringObject;
	```
1. In Python ```x == y``` means content of x and y is equal. but ```x is y``` means id(x) equals to id(y), so to speak the variable x and y is associated to the same object, and address of this object must be equal.
	- ![Pic1](pics/Lecture5_02.png)
	- line 2275-2308 and line 4463-4543 in **Include/ceval.c**: Implementation of instruction ```COMPARE_OP``` in python
	```c
    case COMPARE_OP:
        w = POP();
        v = TOP();
		// if both integers, go inside this.
        if (PyInt_CheckExact(w) && PyInt_CheckExact(v)) {
			...
        }
        else {
          slow_compare:
		  	/* oparg: argument of instruction, "==", ">", "<=" or "is" and so on */
            x = cmp_outcome(oparg, v, w);
        }
		...
        continue;

	static PyObject *
	cmp_outcome(int op, register PyObject *v, register PyObject *w)
	{
	    int res = 0;
		/* switch based on the operation */
	    switch (op) {
	    case PyCmp_IS:
			/* what an 'is' operator does in python is
			   just checking two pointers make sure it's equal,
			   it can be very fast
			*/
	        res = (v == w);
	        break;
	    case PyCmp_IS_NOT:
	        res = (v != w);
	        break;
	    case PyCmp_IN:
	        res = PySequence_Contains(w, v);
	        if (res < 0)
	            return NULL;
	        break;
	    case PyCmp_NOT_IN:
	        res = PySequence_Contains(w, v);
	        if (res < 0)
	            return NULL;
	        res = !res;
	        break;
	    case PyCmp_EXC_MATCH:
			...
	    default:
			/* the most general compare function PyObject_RichCompare(...) defined in Objects/object.c */
	        return PyObject_RichCompare(v, w, op);
	    }
	    v = res ? Py_True : Py_False;
	    Py_INCREF(v);
	    return v;
	}
	```
	- line 593-595 943-979 in **Objects/object.c**: PyObject_RichCompare(...), the most general compare function.
	```c
	/* Macro to get the tp_richcompare field of a type if defined */
	#define RICHCOMPARE(t) (PyType_HasFeature((t), Py_TPFLAGS_HAVE_RICHCOMPARE) \
	             ? (t)->tp_richcompare : NULL)
	/* tp_richcompare is a field in struct PyString_Type(defined in line 3838 of 'Objects/stringobject.c'):
	 * PyTypeObject PyString_Type = {
	 * ...
	 * * function string_richcompare implemented in line 1191 of 'Objects/stringobject.c'
     * (richcmpfunc)string_richcompare,            /* tp_richcompare
	 * ...
	 * };
	 *
	 * definition of function pointer type 'richcmpfunc' in Objects/object.h
	 * typedef PyObject *(*richcmpfunc) (PyObject *, PyObject *, int);
	 */

	PyObject *
	PyObject_RichCompare(PyObject *v, PyObject *w, int op
	{
	    PyObject *res;

	    assert(Py_LT <= op && op <= Py_GE);
	    if (Py_EnterRecursiveCall(" in cmp"))
	        return NULL;

	    /* If the types are equal, and not old-style instances, try to
	       get out cheap (don't bother with coercions etc.). */
	    if (v->ob_type == w->ob_type && !PyInstance_Check(v)) {
	        cmpfunc fcmp;
			/* grab the field 'tp_richcompare'(a function pointer) out from the type object and then apply it */
	        richcmpfunc frich = RICHCOMPARE(v->ob_type);
	        /* If the type has richcmp, try it first.  try_rich_compare
	           tries it two-sided, which is not needed since we've a
	           single type only. */
	        if (frich != NULL) {
	            res = (*frich)(v, w, op);
	            if (res != Py_NotImplemented)
	                goto Done;
	            Py_DECREF(res);
	        }
	        /* No richcmp, or this particular richmp not implemented.
	           Try 3-way cmp. */
	        fcmp = v->ob_type->tp_compare;
	        if (fcmp != NULL) {
	            int c = (*fcmp)(v, w);
	            c = adjust_tp_compare(c);
	            if (c == -2) {
	                res = NULL;
	                goto Done;
	            }
	            res = convert_3way_to_object(op, c);
	            goto Done;
	        }
	    }
	```
	- 流程图总结：
	```viz
	digraph COMPARE_OP {
  	rankdir=LR
    node[shape=record]
	interpreter[label="interpreter(ceval.c) | instruction COMPARE_OP &#92;n function cmp_outcome"];
	object[label = "generic object(object.c) | PyObject_RichCompare(...) &#92;n RICHCOMPARE(v→ob_type)"];
	stringobject[label = "string object(stringobject.c) | PyTypeObject PyString_Type &#92;n field tp_richcompare &#92;n function string_richCompare(...)"];
	interpreter -> object;
	object -> stringobject;
	}
	```

##### Lecture 6 - Code objects, function objects, and closures

1. Objective: To understand how Python functions are simply PyObject structures
	- Code objects
	- Function objects
	- Closures
1. Contents of a function object:
	- foo.func_name: Function name
	- foo.func_dict
	- foo.func_globals: pointer to the global dictionary, when you want to access a global variable in your function code, you can know where to access. As if there is only one global dictionary, why does each function object need a pointer to it? Why don't use just one global pointer that everyone can use it to access global dictionary? Because, if your project has 10 different files, each file would have a different global, each function need to track its own globals.
	- foo.func_code: pointer of a code object, holding the byte coode of the function.
	```viz
	digraph FUNC_OBJ {
		rankdir = LR;
		node [shape=record];
		function_obj[label="Function Object | func_name | <f0>func_globals | <f1>func_code"];
		globals[label="<f0>Global Dictionary | different | from | each | .py file | ... "];
		code_obj[label="Code Object | <f0>co_code | co_consts | co_filename | co_names"]
		bytecode[label="<f0>byte code | ..."];
		function_obj:f0 -> globals:f0;
		function_obj:f1 -> code_obj;
		code_obj:f0 -> bytecode:f0;
	}
	```
1. **NOTE**: the function object isn't created until you actually execute the line of code that define the function. But all the code objects have actually been precompiled, the code object is made during the compilation stage.
1. 代码解析：
	- line 10-30 in **Include/code.h**: Definition of code object, these information can be accessed by ```foo.co_code.xxxx```, the code object doesn't contain any pointer to the global variables.
	```c
	/* Bytecode object */
	typedef struct {
	    PyObject_HEAD
	    int co_argcount;		/* #arguments, except *args */
	    int co_nlocals;		/* #local variables */
	    int co_stacksize;		/* #entries needed for evaluation stack */
	    int co_flags;		/* CO_..., see below */
	    PyObject *co_code;		/* instruction opcodes */
	    PyObject *co_consts;	/* list (constants used) */
	    PyObject *co_names;		/* list of strings (names used) */
	    PyObject *co_varnames;	/* tuple of strings (local variable names) */
	    PyObject *co_freevars;	/* tuple of strings (free variable names) */
	    PyObject *co_cellvars;      /* tuple of strings (cell variable names) */
	    /* The rest doesn't count for hash/cmp */
	    PyObject *co_filename;	/* string (where it was loaded from) */
	    PyObject *co_name;		/* string (name, for reference) */
	    int co_firstlineno;		/* first source line number */
	    PyObject *co_lnotab;	/* string (encoding addr<->lineno mapping) See
					   Objects/lnotab_notes.txt for details. */
	    void *co_zombieframe;     /* for optimization only (see frameobject.c) */
	    PyObject *co_weakreflist;   /* to support weakrefs to code objects */
	} PyCodeObject;
	```
	- line 43-109 in **Objects/codeobject.c**: Constructor of the code object.
	```c
	PyCodeObject *
	PyCode_New(int argcount, int nlocals, int stacksize, int flags,
	           PyObject *code, PyObject *consts, PyObject *names,
	           PyObject *varnames, PyObject *freevars, PyObject *cellvars,
	           PyObject *filename, PyObject *name, int firstlineno,
	           PyObject *lnotab)
	{
	    /* Check argument types */
		...
	    /* Intern selected string constants */
		...
	    co = PyObject_NEW(PyCodeObject, &PyCode_Type);

	    return co;
	}
	```
	- line 472-511 in **Objects/codeobject.c**: Type Object of code object, it contains slots for all functions you can call on the code object.
	```c
	PyTypeObject PyCode_Type = {
	    PyVarObject_HEAD_INIT(&PyType_Type, 0)
	    "code",
		...
    	(cmpfunc)code_compare,              /* tp_compare */
		...
	}
	```
	- line 21-38 in ***Include/funcobject.h***: Definition of the Function Object.
	```c
	/* Function objects and code objects should not be confused with each other:
	 *
	 * Function objects are created by the execution of the 'def' statement.
	 * They reference a code object in their func_code attribute, which is a
	 * purely syntactic object, i.e. nothing more than a compiled version of some
	 * source code lines.  There is one code object per source code "fragment",
	 * but each code object can be referenced by zero or many function objects
	 * depending only on how many times the 'def' statement in the source was
	 * executed so far.
	 */

	typedef struct {
	    PyObject_HEAD
	    PyObject *func_code;	/* A code object */
	    PyObject *func_globals;	/* A dictionary (other mappings won't do) */
	    PyObject *func_defaults;	/* NULL or a tuple(default parameters) */
	    PyObject *func_closure;	/* NULL or a tuple of cell objects */
	    PyObject *func_doc;		/* The __doc__ attribute, can be anything */
	    PyObject *func_name;	/* The __name__ attribute, a string object */
	    PyObject *func_dict;	/* The __dict__ attribute, a dict or NULL */
	    PyObject *func_weakreflist;	/* List of weak references */
	    PyObject *func_module;	/* The __module__ attribute, can be anything */

	    /* Invariant:
	     *     func_closure contains the bindings for func_code->co_freevars, so
	     *     PyTuple_Size(func_closure) == PyCode_GetNumFree(func_code)
	     *     (func_closure may be NULL if PyCode_GetNumFree(func_code) == 0).
	     */
	} PyFunctionObject;
	```
	- line 9-61 in **Objects/funcobject.c**: Function Object = Code Object + Globals
	```c
	PyObject *
	PyFunction_New(PyObject *code, PyObject *globals)
	{
		...
	}
	```
	- line 159-174 and line 332-344 in **Objects/funcobj.c**: Maps python name(like ```func_closure```, ```__closure__```) to actual fields in C struct, to allow python code access directly variables defined in C.
	```c
	#define OFF(x) offsetof(PyFunctionObject, x)

	/* Read Only */
	static PyMemberDef func_memberlist[] = {
	    {"func_closure",  T_OBJECT,     OFF(func_closure),
	     RESTRICTED|READONLY},
	    {"__closure__",  T_OBJECT,      OFF(func_closure),
	     RESTRICTED|READONLY},
	    {"func_doc",      T_OBJECT,     OFF(func_doc), PY_WRITE_RESTRICTED},
	    {"__doc__",       T_OBJECT,     OFF(func_doc), PY_WRITE_RESTRICTED},
	    {"func_globals",  T_OBJECT,     OFF(func_globals),
	     RESTRICTED|READONLY},
	    {"__globals__",  T_OBJECT,      OFF(func_globals),
	     RESTRICTED|READONLY},
	    {"__module__",    T_OBJECT,     OFF(func_module), PY_WRITE_RESTRICTED},
	    {NULL}  /* Sentinel */
	};
	/* Read and Set */
	static PyGetSetDef func_getsetlist[] = {
	    {"func_code", (getter)func_get_code, (setter)func_set_code},
	    {"__code__", (getter)func_get_code, (setter)func_set_code},
	    {"func_defaults", (getter)func_get_defaults,
	     (setter)func_set_defaults},
	    {"__defaults__", (getter)func_get_defaults,
	     (setter)func_set_defaults},
	    {"func_dict", (getter)func_get_dict, (setter)func_set_dict},
	    {"__dict__", (getter)func_get_dict, (setter)func_set_dict},
	    {"func_name", (getter)func_get_name, (setter)func_set_name},
	    {"__name__", (getter)func_get_name, (setter)func_set_name},
	    {NULL} /* Sentinel */
	};
	/* line 547-586: definition of Type Object of the function object */
	PyTypeObject PyFunction_Type = {
	    PyVarObject_HEAD_INIT(&PyType_Type, 0)
	    "function",
		...
		/*function_call is used to call the function*/
    	function_call,                              /* tp_call */
		...
	}
	/* line 487-524 definition of the method function_call */
	static PyObject *
	function_call(PyObject *func, PyObject *arg, PyObject *kw)
	{
		...
		/* function PyEval_EvalCodeEx actually get all information
		 * out from the Function Object and create the frame then
		 * pass to the interpreter main loop function
		 * PyEval_EvalFrameEx to execute the function
		 * 与前面CALL_FUNCTION指令中调用的fast_function应用场景
		 * 有何不同？
		 */
    	result = PyEval_EvalCodeEx(
    	    (PyCodeObject *)PyFunction_GET_CODE(func),
    	    PyFunction_GET_GLOBALS(func), (PyObject *)NULL,
    	    &PyTuple_GET_ITEM(arg, 0), PyTuple_GET_SIZE(arg),
    	    k, nk, d, nd,
    	    PyFunction_GET_CLOSURE(func));

    	return result;
	}
	```
1. Closure
    ![Pic1](pics/Lecture6_01.png)
    ![Pic1](pics/Lecture6_02.png)
1. Source Code about instruction LOAD_DEREF
	- line 2157-2181 in **Python/ceval.c**
	```c
        case LOAD_DEREF:
			/* get name of the environment variables, can get by:
			 * test3.b2.func_code.co_freevars
			 */
            x = freevars[oparg];
            w = PyCell_Get(x);
			...
	```
	- line 26 and line 33-37 in **Include/funcobject.h**
	```c
    PyObject *func_closure;	/* NULL or a tuple of cell objects */

    /* Invariant:
     *     func_closure contains the bindings for func_code->co_freevars, so
     *     PyTuple_Size(func_closure) == PyCode_GetNumFree(func_code)
     *     (func_closure may be NULL if PyCode_GetNumFree(func_code) == 0).
     */
	```

##### Lecture 7 - Iterators

1. Objective: To understand how iterators and 'for' loops work under the hood.
	- Python 'for' loops over a list: ceval.c, abstract.c, listobject.c
	- Defining a class with a custom iterator: iterator.h, iterobject.c
1. Iterator: A object that encapsulate the act of iterating. Need have one method called ```next()``` to get the next elememt of iterating.
	- To make a iterator from a list: ```i = iter(lst)``` or ```i = lst.__iter__()```, then the ```i``` is a iterator object, it's not actually the ```lst```, but it holds a pointer(the pointer can only advance forward, can't go backwards, and iterators in python don't support modification) points to the first element of the list. Everytime call the ```next()``` method on ```i```, will return the current element of the pointer and advance the pointer to the next element in the list. At the very end of the list, call the ```next()``` function will raise a ```StopIteration``` exception.
	- Why do we need iterator instead of accessing elements by index directly?
		1. You can make code shorter by iterator.
		1. You don't need to worry about bound of the index.
		1. You can define a iterator for a complex data struct(like a tree) and make it retrieve data in the struct in a determined order without worry about the index and its bound.
	- Python code of using a iterator to print all elements in a list:
	```python
	lst = ['a', 'b', 'c']
	i = lst.__iter__() # 或者 i = iter(lst)
	while True:
	    try:
	        print i.next()
	    except StopIteration:
	        break
	# This for loop implicitly make an iterator to iterate on the list
	# Code behind a for loop is kind of similar to code above
	for elt in lst:
	    print elt
	```
	- Using a for loop with builtin iterator to print elements in a dictionary you can only get the keys out in a undeterminaed order.
	- A ```String``` object also has a builtin iterator that can put  characters in the string out orderly while calling ```next()``` on it.
1. Instructions of 'for' loop:
	- ![Pic1](pics/Lecture7_01.png)
1. Diving into the code of ```GET_ITER``` and ```FOR_ITER```:
	- line 2493-2528 in **Python/ceval.c**
	```c
    case GET_ITER:
        /* before: [obj]; after [getiter(obj)] */
        v = TOP();
        x = PyObject_GetIter(v);
        Py_DECREF(v);
        if (x != NULL) {
            SET_TOP(x);
	        /* as GET_ITER is always followed by FOR_ITER,
	         * we can jump to FOR_ITER without go back to the
	         * main interpreter loop
	         */
            PREDICT(FOR_ITER);
            continue;
        }
        STACKADJ(-1);
        break;

    PREDICTED_WITH_ARG(FOR_ITER);
    case FOR_ITER:
        /* before: [iter]; after: [iter, iter()] *or* [] */
        v = TOP();
        /* call the next() method on the iterator object */
        x = (*v->ob_type->tp_iternext)(v);
        if (x != NULL) {
            PUSH(x);
	        /* push the value gotten back to the stack */
            PREDICT(STORE_FAST);
            PREDICT(UNPACK_SEQUENCE);
	        /* go back to the main iterpreter loop and execute the next opcode */
            continue;
        }
        if (PyErr_Occurred()) {
            if (!PyErr_ExceptionMatches(
                            PyExc_StopIteration))
                break;
            PyErr_Clear();
        }
        /* iterator ended normally
	     * once x == NULL and no exception happend, we'll come here
         * jump to the instruction of POP_BLOCK and then go back to the 
	     */
        x = v = POP();
        Py_DECREF(v);
        // #define JUMPBY(x)       (next_instr += (x))
        JUMPBY(oparg);
        continue;

	```
	- line 3066-3090 in **Objects/abstract.c**: definition of the function ```PyObject_GetIter``` - how to get a iterator from a generic object, used by struction ```GET_ITER```. There may be 2 kinds of iterators in Python generic iterator defined by the Type Object and the sequence Iterator for Sequence Objects.
	```c
    PyObject *
    PyObject_GetIter(PyObject *o)
    {
        PyTypeObject *t = o->ob_type;
        getiterfunc f = NULL;
	    /* if type of the object has a flag py_TPFLAGS_HAVE_ITER
	     * grab the tp_iter method from the type object
	     */
        if (PyType_HasFeature(t, Py_TPFLAGS_HAVE_ITER))
            f = t->tp_iter;
        if (f == NULL) {
	        /* if there is no tp_iter method, see if it's a sequence */
            if (PySequence_Check(o))
                /* if it's a sequence object, create a sequence iterator
	             * from the sequence and then return it
                 */
                return PySeqIter_New(o);
            return type_error("'%.200s' object is not iterable", o);
        }
        else {
            PyObject *res = (*f)(o);
            if (res != NULL && !PyIter_Check(res)) {
                PyErr_Format(PyExc_TypeError,
                             "iter() returned non-iterator "
                             "of type '%.100s'",
                             res->ob_type->tp_name);
                Py_DECREF(res);
                res = NULL;
            }
            return res;
        }
    }
	```
	- line 11-28 in **Objects/iterobject.c**: definition of the function PySeqIter_New - how to create an iterator for a sequence object.
	```c
    PyObject *
    PySeqIter_New(PyObject *seq)
    {
        seqiterobject *it;

        if (!PySequence_Check(seq)) {
            PyErr_BadInternalCall();
            return NULL;
        }
	    /* create the iterator object */
        it = PyObject_GC_New(seqiterobject, &PySeqIter_Type);
        if (it == NULL)
            return NULL;
        /* set index of the iterator 0 */
        it->it_index = 0;
        Py_INCREF(seq);
	    /* keep the original sequence object in the iterator for track */
        it->it_seq = seq;
        _PyObject_GC_TRACK(it);
        return (PyObject *)it;
    }
	/* line 5-9 in Objects/iterobject.c: definition of a sequence iterator object */
    typedef struct {
        PyObject_HEAD
        long      it_index;
        PyObject *it_seq; /* Set to NULL when iterator is exhausted */
    } seqiterobject;
	/* line 96-127 in Objects/iterobject.c:
	 * definition of Type Object of sequence iterator object
	 */
    PyTypeObject PySeqIter_Type = {
        PyVarObject_HEAD_INIT(&PyType_Type, 0)
        "iterator",                                 /* tp_name */
	    ...
	    /* method to get next element from the iterator */
        iter_iternext,                              /* tp_iternext */
	    ...
	}
	/* line 45-71 in Object/iterobject.c: implemention of function iter_iternext */
    static PyObject *
    iter_iternext(PyObject *iterator)
    {
        seqiterobject *it;
        PyObject *seq;
        PyObject *result;

        assert(PySeqIter_Check(iterator));
        it = (seqiterobject *)iterator;
		/* get the sequence object */
        seq = it->it_seq;
        if (seq == NULL)
            return NULL;

        /* get the element with it_inddex to return */
        result = PySequence_GetItem(seq, it->it_index);
        if (result != NULL) {
			/* increase the index by 1: core part of the iterating */
            it->it_index++;
            return result;
        }
        if (PyErr_ExceptionMatches(PyExc_IndexError) ||
            PyErr_ExceptionMatches(PyExc_StopIteration))
        {
            PyErr_Clear();
            Py_DECREF(seq);
            it->it_seq = NULL;
        }
        return NULL;
    }
	```
	- About the generic(non-sequence object) iterator in python
		- ![Pic1](pics/Lecture7_02.png)

##### Lecture 8 - User-defined classes and objects

1. Objective: To understand how Python implements classes and objects
	- Classes and objects in Python
	- Include/classobject.h, Objects/classobject.c
1. Overview:
	- ![Pic1](pics/Lecture8_01.png)
	- ![Pic1](pics/Lecture8_02.png)
1. Source Code
	- line 1936-1946 in **Python/ceval.c**: about the instruction BUILD_CLASS
	```c
    case BUILD_CLASS:
	    /* get the method dictionary */
        u = TOP();
	    /* get the tuple of the names of the base classes */
        v = SECOND();
	    /* get the class name */
        w = THIRD();
        STACKADJ(-2);
	    /* build the class object */
        x = build_class(u, v, w);
	    /* push the new class object into the stack */
        SET_TOP(x);
        Py_DECREF(u);
        Py_DECREF(v);
        Py_DECREF(w);
        break;
	```
	- line 4618-4670 in **Python/ceval.c**: definition of function ```build_class(..)```
	```c
    static PyObject *
    build_class(PyObject *methods, PyObject *bases, PyObject *name)
    {
	    ...
	    /* Just image metaclass as a function that
	     * take name, bases, and methods to build a
	     * class object. Interpreter will call
	     * a default metaclass to create a class
	     * without a specific metaclass.
	     */
        result = PyObject_CallFunctionObjArgs(metaclass, name, bases, methods, NULL);
	    ...
	    return result
	}
	```
	- line 12-37 in **Include/classobject.h**: Data struct associated with Class Object and Instance of Class.
	```c
	/* Class Object */
    typedef struct {
        PyObject_HEAD
        PyObject	*cl_bases;	/* A tuple of class objects */
        PyObject	*cl_dict;	/* A dictionary */
        PyObject	*cl_name;	/* A string */
        /* The following three are functions or NULL */
        PyObject	*cl_getattr;
        PyObject	*cl_setattr;
        PyObject	*cl_delattr;
        PyObject    *cl_weakreflist; /* List of weak references */
    } PyClassObject;

    /* Instance Object: instance doesn't store any methods itself */
    typedef struct {
        PyObject_HEAD
	    /* Pointer to class of the instance to get methods(only stored in class) */
        PyClassObject *in_class;	/* The class object */
	    /* values of fields in the instance */
        PyObject	  *in_dict;	/* A dictionary */
        PyObject	  *in_weakreflist; /* List of weak references */
    } PyInstanceObject;

	/* Method Object, Method is just a function wrapped with
	 * a 'self' pointer and a 'class' pointer
	 */
    typedef struct {
        PyObject_HEAD
	    /* the function */
        PyObject *im_func;   /* The callable object implementing the method */
		/* the specific instance that calls the method */
        PyObject *im_self;   /* The instance it is bound to, or NULL */
        PyObject *im_class;  /* The class that asked for the method */
        PyObject *im_weakreflist; /* List of weak references */
    } PyMethodObject;
	```
	- Relation between Method and Function
		![Pic1](pics/Lecture8_03.png)
	- Every Instance stores a pointer to its class to get methods(metods are only  stored in the class object) When a instance call a method it will transfer itself as the 'self' parameter to the metod, so that even different instance is excuting the same method in the class, it won't mess up.
	- Process of creating instance of a class:
	```viz
	digraph instance {
		node [shape = record];
		initial[label="Call the class name like a function in python code: Counter(5, 7)"]
		bytecode[label="Compiled into instruction CALL_FUNCTION Counter with parameter 5 and 7"]
		call_function[label="By definition of Instruction CALL_FUNCTION invoke the function call_function(...) in ceval.c"]
		do_call[label="as it's a Class Object instead a REAL function, we'll invoke the function do_call(...) in ceval.c"]
		PyObject_Call[label="invoke the function PyObject_Call(...) definded in abstract.c"]
		tp_call[label="invoke field tp_call(PyInstance_New) of struct PyClass_Type in classobject.c "]
		instance_new[label="function PyInstance_New executed and create the Instance Object finally"]
		initial->bytecode;
		bytecode->call_function
		call_function->do_call
		do_call->PyObject_Call
		PyObject_Call->tp_call
		tp_call->instance_new
	}
	```
	- Source Code associated with creation of Insatance
	```c
	/* line 444-484 in Objects/classobject.c:
	 * definition of PyClass_Type - Type Object of the class object
	 */
    PyTypeObject PyClass_Type = {
        PyObject_HEAD_INIT(&PyType_Type)
        0,
        "classobj",
        sizeof(PyClassObject),
    	...
        /* called by Instruction CALL_FUNCTION
         * following the process described above
	     * when instantiaze a Class
         */
        PyInstance_New,                             /* tp_call */
    	...
    };
	/* line 549-598 in Objects/classobject.c:
	 * implementation of function PyInsatnce_NEW
	 */
    PyObject *
    PyInstance_New(PyObject *klass, PyObject *arg, PyObject *kw)
    {
        register PyInstanceObject *inst;
        PyObject *init;
        static PyObject *initstr;

        if (initstr == NULL) {
            initstr = PyString_InternFromString("__init__");
            if (initstr == NULL)
                return NULL;
        }
	    /* create a unintialized Instance first */
        inst = (PyInstanceObject *) PyInstance_NewRaw(klass, NULL);
        /* get the attribute 'init' from the instance*/
        init = instance_getattr2(inst, initstr);
        if (init == NULL) {
		    ...
		    return null
		} else {
	        /* Apply Method Object 'init' on the new instance to initialize it
	         * with arguments given by Customer code, like Counter(5, 7)
	         *
	         * As 'init' is a Method Object and keeps a pointer(im_self) to the
	         * instance('inst') itself, function PyEval_CallObjectWithKeyword
	         * doesn't need get 'inst' as a parameter to modify the 'inst'
	         *
	         * 'res' is return value of the __init__() method, should be None
	         */
            PyObject *res = PyEval_CallObjectWithKeywords(init, arg, kw);
	        ...
		}
        return (PyObject *)inst;
	}
	```

##### Lecture 9 - Generators

没有这个↓↓↓在markdown_preview_enhanced中预览前面的mermaid图都会乱(但是在Chrome中打开后显示正常)，不知为何

```mermaid
graph LR
	a;

```

```viz
digraph G {
  rankdir=LR
  node [shape=plaintext]
  a [
     label=<
<TABLE BORDER="0" CELLBORDER="1" CELLSPACING="0">
  <TR><TD ROWSPAN="3" BGCOLOR="yellow">class</TD></TR>
  <TR><TD PORT="here" BGCOLOR="lightblue">qualifier</TD></TR>
</TABLE>>
  ]
    b [shape=ellipse style=filled
  label=<
<TABLE BGCOLOR="bisque">
  <TR><TD COLSPAN="3">elephant</TD> 
      <TD ROWSPAN="2" BGCOLOR="chartreuse" 
          VALIGN="bottom" ALIGN="right">two</TD> </TR>
  <TR><TD COLSPAN="2" ROWSPAN="2">
        <TABLE BGCOLOR="grey">
          <TR> <TD>corn</TD> </TR> 
          <TR> <TD BGCOLOR="yellow">c</TD> </TR> 
          <TR> <TD>f</TD> </TR> 
        </TABLE> </TD>
      <TD BGCOLOR="white">penguin</TD> 
  </TR> 
  <TR> <TD COLSPAN="2" BORDER="4" ALIGN="right" PORT="there">4</TD> </TR>
</TABLE>>
  ]
  c [ 
  label=<long line 1<BR/>line 2<BR ALIGN="LEFT"/>line 3<BR ALIGN="RIGHT"/>>
  ]

  subgraph { rank=same b c }
  a:here -> b:there [dir=both arrowtail = diamond]
  c -> b
  d [shape=triangle]
  d -> c [label=<
<TABLE>
  <TR><TD BGCOLOR="red" WIDTH="10"> </TD>
      <TD>Edge labels<BR/>also</TD>
      <TD BGCOLOR="blue" WIDTH="10"> </TD>
  </TR>
</TABLE>>
  ]
}
```

```viz
digraph structs {
    node [shape=record];
    struct1 [label="<f0> left|<f1> mid&#92; dle|<f2> right"];
    struct2 [label="<f0> one|<f1> two"];
    struct3 [label="hello&#92;nworld |{ b |{c|<here> d|e}| f}| g | h"];
    struct1:f1 -> struct2:f0;
    struct1:f2 -> struct3:here;
}
```

```viz
digraph structs {
    node [shape=plaintext]
    struct1 [label=<
<TABLE BORDER="0" CELLBORDER="1" CELLSPACING="0">
  <TR><TD>left</TD><TD PORT="f1">mid dle</TD><TD PORT="f2">right</TD></TR>
</TABLE>>];
    struct2 [label=<
<TABLE BORDER="0" CELLBORDER="1" CELLSPACING="0">
  <TR><TD PORT="f0">one</TD><TD>two</TD></TR>
</TABLE>>];
    struct3 [label=<
<TABLE BORDER="0" CELLBORDER="1" CELLSPACING="0" CELLPADDING="4">
  <TR>
    <TD ROWSPAN="3">hello<BR/>world</TD>
    <TD COLSPAN="3">b</TD>
    <TD ROWSPAN="3">g</TD>
    <TD ROWSPAN="3">h</TD>
  </TR>
  <TR>
    <TD>c</TD><TD PORT="here">d</TD><TD>e</TD>
  </TR>
  <TR>
    <TD COLSPAN="3">f</TD>
  </TR>
</TABLE>>];
    struct1:f1 -> struct2:f0;
    struct1:f2 -> struct3:here;
}
```

```viz
digraph dfd{
    node[shape=record]
    store1 [label="{<f0> left|<f1> Some data store}"];
    proc1 [label="{<f0> 1.0|<f1> Some process here\n\n\n}" shape=Mrecord];
    enti1 [label="Customer" shape=box];
    store1:f0 -> proc1:f1;
    enti1-> proc1:f0;
}
```

VIM 快捷键：
首先，可以在命令模式下输入v进入自由选取模式，选择需要剪切的文字后，按下d就可以进行剪切了。
其他命令模式下剪切命令：
dd：剪切当前行
ndd：n表示大于1的数字，剪切n行
dw：从光标处剪切至一个单子/单词的末尾，包括空格
de：从光标处剪切至一个单子/单词的末尾，不包括空格
d$：从当前光标剪切到行末
d0：从当前光标位置（不包括光标位置）剪切之行首
d3l：从光标位置（包括光标位置）向右剪切3个字符
d5G：将当前行（包括当前行）至第5行（不包括它）剪切
d3B：从当前光标位置（不包括光标位置）反向剪切3个单词
dH：剪切从当前行至所显示屏幕顶行的全部行
dM：剪切从当前行至命令M所指定行的全部行
dL：剪切从当前行至所显示屏幕底的全部行

yy：复制当前行
nyy：n表示大于1的数字，复制n行
yw：从光标处复制至一个单子/单词的末尾，包括空格
ye：从光标处复制至一个单子/单词的末尾，不包括空格
y$：从当前光标复制到行末
y0：从当前光标位置（不包括光标位置）复制之行首
y3l：从光标位置（包括光标位置）向右复制3个字符
y5G：将当前行（包括当前行）至第5行（不包括它）复制
y3B：从当前光标位置（不包括光标位置）反向复制3个单词