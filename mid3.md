3번 문제
```
#include <stdio.h>
int main(){
    int a[3], b[3], s = 0, bl = 0;
    for(int i = 0; i < 3; i++) scanf("%d", &a[i]);
    for(int i = 0; i < 3; i++) scanf("%d", &b[i]);
    for(int i = 0; i < 3; i++)
        for(int j = 0; j < 3; j++)
            if(a[i] == b[j]){ if(i == j) s++; else bl++; }
    printf("%d Strike, %d Ball\n", s, bl);
    if(s == 3) printf("HOMERUN\n");
    return 0;
}
```
