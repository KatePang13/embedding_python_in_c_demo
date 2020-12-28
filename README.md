# 将python嵌入C语言

## 导入python模块

- 导入 python 模块  `pModule = PyImport_Import(moduleName); `
- 获取函数地址    `pFunc = PyObject_GetAttrString(pModule, funcName);`
- 调用函数            `ret = PyObject_CallObject(pFunc, args);  `

### 示例代码

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>

int
main(int argc, char *argv[])
{
    PyObject *pName, *pModule, *pFunc;
    PyObject *pArgs, *pValue;
    int i;

    if (argc < 3) {
        fprintf(stderr,"Usage: call pythonfile funcname [args]\n");
        return 1;
    }

    Py_Initialize();
    pName = PyUnicode_DecodeFSDefault(argv[1]);
    /* Error checking of pName left out */

    pModule = PyImport_Import(pName);
    Py_DECREF(pName);

    if (pModule != NULL) {
        pFunc = PyObject_GetAttrString(pModule, argv[2]);
        /* pFunc is a new reference */

        if (pFunc && PyCallable_Check(pFunc)) {
            pArgs = PyTuple_New(argc - 3);
            for (i = 0; i < argc - 3; ++i) {
                pValue = PyLong_FromLong(atoi(argv[i + 3]));
                if (!pValue) {
                    Py_DECREF(pArgs);
                    Py_DECREF(pModule);
                    fprintf(stderr, "Cannot convert argument\n");
                    return 1;
                }
                /* pValue reference stolen here: */
                PyTuple_SetItem(pArgs, i, pValue);
            }
            pValue = PyObject_CallObject(pFunc, pArgs);
            Py_DECREF(pArgs);
            if (pValue != NULL) {
                printf("Result of call: %ld\n", PyLong_AsLong(pValue));
                Py_DECREF(pValue);
            }
            else {
                Py_DECREF(pFunc);
                Py_DECREF(pModule);
                PyErr_Print();
                fprintf(stderr,"Call failed\n");
                return 1;
            }
        }
        else {
            if (PyErr_Occurred())
                PyErr_Print();
            fprintf(stderr, "Cannot find function \"%s\"\n", argv[2]);
        }
        Py_XDECREF(pFunc);
        Py_DECREF(pModule);
    }
    else {
        PyErr_Print();
        fprintf(stderr, "Failed to load \"%s\"\n", argv[1]);
        return 1;
    }
    if (Py_FinalizeEx() < 0) {
        return 120;
    }
    return 0;
}
```

### 编译&链接&执行

这里要以具体安装的python版本为准

```shell
#编译
gcc -o call.o -c call.c $(python3.6m-config --cflags)

#链接
gcc -o call call.o  $(python3.6m-config --ldflags)

#执行
./call math pow 5 2
Result of call: 25
```



## 导出python模块

- 定义方法 `static PyOBject* func(PyObject *self, PyObject *args)`
- 定义方法列表  `static PyMethodDef EmbMethods[]`
- 定义模块 `static PyModuleDef EmbModule`
- 创建 module 对象   `  embModuleObj = PyModule_Create(&EmbModule`) `
- 将 模块 导入 到 嵌入python解释器中 `PyImport_AppendInittab("emb", &embModuleObj );`

## 示例代码

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>

static int numargs=0;

/* Return the number of arguments of the application command line */
static PyObject* emb_numargs(PyObject *self, PyObject *args)
{
    if(!PyArg_ParseTuple(args, ":numargs"))
        return NULL;
    return PyLong_FromLong(numargs);
}

//声明 module emb 方法列表
static PyMethodDef EmbMethods[] = {
    {"numargs", emb_numargs, METH_VARARGS, "Return the number of arguments received by the process."},
    {NULL, NULL, 0, NULL}
};

//声明 module: emb 
static PyModuleDef EmbModule = {
    PyModuleDef_HEAD_INIT, "emb", NULL, -1, EmbMethods,
    NULL, NULL, NULL, NULL
};

//创建 module 对象
static PyObject* PyInit_emb(void)
{
    return PyModule_Create(&EmbModule);
}

int main(int argc, char *argv[])
{
    PyObject *pName, *pModule, *pFunc;
    PyObject *pArgs, *pValue;
    int i;

    if (argc < 3) {
        fprintf(stderr,"Usage: call pythonfile funcname [args]\n");
        return 1;
    }
    
    numargs = argc;
	PyImport_AppendInittab("emb", &PyInit_emb);//使 emb.numargs 在嵌套的python解释器中可用

    Py_Initialize();
    pName = PyUnicode_DecodeFSDefault(argv[1]);
    /* Error checking of pName left out */

    pModule = PyImport_Import(pName);
    Py_DECREF(pName);

    if (pModule != NULL) {
        pFunc = PyObject_GetAttrString(pModule, argv[2]);
        /* pFunc is a new reference */

        if (pFunc && PyCallable_Check(pFunc)) {
            pArgs = PyTuple_New(argc - 3);
            for (i = 0; i < argc - 3; ++i) {
                pValue = PyLong_FromLong(atoi(argv[i + 3]));
                if (!pValue) {
                    Py_DECREF(pArgs);
                    Py_DECREF(pModule);
                    fprintf(stderr, "Cannot convert argument\n");
                    return 1;
                }
                /* pValue reference stolen here: */
                PyTuple_SetItem(pArgs, i, pValue);
            }
            pValue = PyObject_CallObject(pFunc, pArgs);
            Py_DECREF(pArgs);
            if (pValue != NULL) {
                printf("Result of call: %ld\n", PyLong_AsLong(pValue));
                Py_DECREF(pValue);
            }
            else {
                Py_DECREF(pFunc);
                Py_DECREF(pModule);
                PyErr_Print();
                fprintf(stderr,"Call failed\n");
                return 1;
            }
        }
        else {
            if (PyErr_Occurred())
                PyErr_Print();
            fprintf(stderr, "Cannot find function \"%s\"\n", argv[2]);
        }
        Py_XDECREF(pFunc);
        Py_DECREF(pModule);
    }
    else {
        PyErr_Print();
        fprintf(stderr, "Failed to load \"%s\"\n", argv[1]);
        return 1;
    }
    if (Py_FinalizeEx() < 0) {
        return 120;
    }
    return 0;
}
```

### 编译&链接&执行

这里要以具体安装的python版本为准

```shell
#编译
gcc -o call.o -c call.c $(python3.6m-config --cflags)

#链接
gcc -o call call.o  $(python3.6m-config --ldflags)

#执行
./call emb numargs
Result of call: 3
```





## Reference:

https://docs.python.org/3/extending/embedding.html

https://stackoverflow.com/questions/27672572/embedding-python-in-c-linking-fails-with-undefined-reference-to-py-initialize