
# Главное различие в том, что при передаче по указателю в функцию в Golang - указатель копируется!!! Те создается локальная переменная, которая содержит адрес на изначальную переменную. Чтобы модифицировать изначальную переменную, нужно разименовать этот указатель. В Golang НЕТ!!! передачи по ссылке. 
### Passing by Pointer

When you pass a pointer to a function, you are passing the address of a variable. This allows the function to modify the value of the variable at that address. Here's an example:
```c
#include <stdio.h>

void increment(int *num) {
    (*num)++;
}

int main() {
    int a = 5;
    increment(&a);
    printf("Value of a: %d\n", a);  // Output will be 6
    return 0;
}

```
- The `increment` function takes a pointer to an integer (`int *num`).
- The `&a` syntax in `main` passes the address of `a` to the function.
- The function uses `(*num)++` to dereference the pointer and increment the value at that address.

### Passing by Reference

In languages that do, such as C++, you can pass a reference to a variable, allowing the function to operate directly on the original variable without using pointers explicitly. In C++, the equivalent would be:
```cpp
#include <iostream>
using namespace std;

void increment(int &num) {
    num++;
}

int main() {
    int a = 5;
    increment(a);
    cout << "Value of a: " << a << endl;  // Output will be 6
    return 0;
}

```
- The `increment` function takes a reference to an integer (`int &num`).
- No need for dereferencing inside the function. `num++` directly increments the original variable.