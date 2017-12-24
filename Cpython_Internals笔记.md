## <p align="center"> Cpython Internals笔记 </p>
##### Lecture 1 - Interpreter and source code overview
1. Interpreter and source code overview.
```mermaid
graph LR
   srcode[source code]==>compiler((compiler:</br>Standard));
   compiler==>bytecode[bytecode]
   bytecode==>interpreter(interpreter);
   interpreter==>output;
   subgraph 重点关注
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
	*可用byteplay包替代上述命令获得更好的反编译效果

2. 代码注释
	1. ceval.c
		- ines 693-3021: main interpreter, takes a frame, returns a python object.
		```cpp
		PyObject *PyEval_EvalFrameEx(PyFrameObject *f, int throwflag){
		// line 689-730: locale variables storethe locale states
		// line 698: The pointer to the value stack
		register PyObject **stack_pointer; /* Next free slot in value stack */
		// line 825-858: stack manipulation macros
		// line 919-945: grab everything out from code(frame)
		// line 964: a giant infinite loop: go through the bytecode one byte a time
		for (;;) {
			// line 1080: Extract the next opcode
			opcode = NEXTOP();
			// line 1083: Extract args if have arg
			if (HAS_ARG(opcode))
            	oparg = NEXTARG();
			}
			// line 1112: Execute the opcode by switch
			switch (opcode) {
				CASE LOAD_FAST: ... break;
				CASE LOAD_CONST: ... break;
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

##### Lecture 4 - PyObject: The core Python object

##### Lecture 5 - Example Python data types

##### Lecture 6 - Code objects, function objects, and closures

##### Lecture 7 - Iterators

##### Lecture 8 - User-defined classes and objects

##### Lecture 9 - Generators
