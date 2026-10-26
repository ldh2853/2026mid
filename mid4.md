4번 문제
```
#include <stdio.h>

int main(void)
{
    int n;
    scanf("%d", &n);
    for (int i = 1; i <= n; i++) {
        for (int j = 0; j < n - i; j++) putchar(' ');
        for (int j = 0; j < 2 * i - 1; j++) putchar(i % 2 ? '*' : '#');
        putchar('\n');
    }
    return 0;
}
```
