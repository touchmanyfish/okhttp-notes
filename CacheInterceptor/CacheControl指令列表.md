## request指令
```text
max-age（处理了）

max-stale （处理了）

min-fresh （处理了）

no-cache （处理了）

no-store（处理了）

no-transform 没处理

only-if-cached 处理了
```

## response指令
```text
max-age 处理了

s-maxage(okhttp中cache是private的，所以直接ignore这个指令)

no-cache(处理了)

no-store（处理了）

no-transform

public（??）

private(??)

must-revalidate（剔除max-stale）

proxy-revalidate（效果和must-revalidate一样，但是private cache不能使用它）

must-understand（注：由 RFC 9111 明确，但在 7234 体系的扩展中常被提及
```