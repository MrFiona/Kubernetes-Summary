# YAML文件

## YAML基础

它的基本语法规则如下：

- 大小写敏感
- 使用缩进表示层级关系
- 缩进时不允许使用Tab键，只允许使用空格。
- 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可
- `#`表示注释，从这个字符一直到行尾，都会被解析器忽略。

在我们的 kubernetes 中，你只需要两种结构类型就行了：
- Lists
- Maps

也就是说，你可能会遇到 Lists 的 Maps 和 Maps 的 Lists，等等。不过不用担心，你只要掌握了这两种结构也就可以了，其他更加复杂的我们暂不讨论。


## Maps

首先我们来看看 Maps，我们都知道 Map 是字典，就是一个key:value的键值对，Maps 可以让我们更加方便的去书写配置信息，例如：

```
---
apiVersion: v1
kind: Pod

```

第一行的---是分隔符，是可选的，在单一文件中，可用连续三个连字号---区分多个文件。这里我们可以看到，我们有两个键：kind 和 apiVersion，他们对应的值分别是：v1 和Pod。上面的 YAML 文件转换成 JSON 格式的话，你肯定就容易明白了

```
{
    "apiVersion": "v1",
    "kind": "pod"
}

```

我们在创建一个相对复杂一点的 YAML 文件，创建一个 KEY 对应的值不是字符串而是一个 Maps：

```
---
apiVersion: v1
kind: Pod
metadata:
  name: kube100-site
  labels:
    app: web

```
