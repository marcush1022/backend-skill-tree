# **json 包变量不加 tag 会怎么样？**

## **问题**
- json 包里使用的时候，结构体里的变量不加 tag 能不能正常转成 json 里的字段？

## **怎么答**
- **如果变量首字母`小写`，则为 `private`。无论如何不能转，因为取不到反射信息**。

- **如果变量首字母`大写`，则为 `public`**。
    - **不加 tag，可以正常转为 json 里的字段，json 内字段名跟`结构体内字段原名`一致**。

    - **加了 tag，从 struct 转 json 的时候，json 的字段名就是 `tag 里的字段名`，原字段名已经没用**。

## **举例**
- **通过一个例子加深理解**

    ```go
    package main
    import (
        "encoding/json"
        "fmt"
    )
    type J struct {
        a string             //小写无tag
        b string `json:"B"`  //小写+tag
        C string             //大写无tag
        D string `json:"DD"` //大写+tag
    }
    func main() {
        j := J {
        a: "1",
        b: "2",
        C: "3",
        D: "4",
        }
        fmt.Printf("转为json前j结构体的内容 = %+v\n", j)
        jsonInfo, _ := json.Marshal(j)
        fmt.Printf("转为json后的内容 = %+v\n", string(jsonInfo))
    }
    ```

- **输出**
    - 转为 json 前结构体的内容 = `{a:1 b:2 C:3 D:4}`
    
    - 转为 json 后的内容 = `{"C":"3","DD":"4"}`

- **解释**
    - 结构体里定义了四个字段，分别对应**小写无 tag，小写 + tag，大写无 tag，大写 + tag**。

    - **转为 json 后首字母`小写的`不管加不加 tag 都`不能`转为 json 里的内容**
    
    - **而`大写的`加了 tag 可以取`别名`，不加 tag 则 json 内的字段跟`结构体字段原名`一致**。