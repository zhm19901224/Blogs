### 回顾：JavaScript所有与正则相关的方法(含es6新特性)

#### 回顾常用的正则的符号
> 匹配哪些字符？
1. [abc] 匹配abc中任意一个字符都行
2. [^abc] 匹配除了这三个字母意外的任何字符
3. (abc) 匹配连续完整的abc 用于分组
4. (abc|def) 匹配连续完整的abc或者def
5. [A-Za-z0-9_]   \w   匹配字母数字下划线
6. [\s\S] 匹配所有空格和换行
7.  .任意字符（utf16、\r\n行终止符不行）
8. \b 单词之间的空格
9. $匹配头，^匹配末尾

> 匹配到的字符，要匹配多少个?
1. {2, 8}： 匹配2-8个前面的字符，如果数量不满足，就会忽略
2. *： 0次或多次，可有可无
3. ?： 0次或1次
4. +： 1次或多次

> 匹配模式
1. g全局匹配
2. i不区分大小写匹配
#### 1. RegExt.prototype.test(str)
> 返回boolean，判断字符串是否整体与正则匹配，或者是否存在一部分与正则表达式匹配
```
// 部分匹配情况
let reg1 = /abc*.*def*/

let str = '111abc222def333'

console.log(reg1.test(str))     // true

// 整体匹配
let reg2 = /^111.*333$/

console.log(reg2.test(str))     // true

```


#### 2. RegExt.prototype.exec()
> 在没有子表达式情况下，用法和match一样, match处理不了子表达式，而exec能返回包含子表达式的结果。
> 内部有迭代器，妹执行完一次，再次执行，匹配的字符串是前一次匹配完的剩余部分。

```
let str = "3abc4,5abc6";
let reg = /a(bc)/g

console.log(reg.exec(str));     // [ 'abc', 'bc', index: 1, input: '3abc4,5abc6', groups: undefined ]
console.log(str.match(reg));    // [ 'abc', 'abc' ]
```



#### 3. String.prototype.search(Reg)
> 返回找到 第一个 满足正则表达式的 子串 的 索引，没找到返回-1

```
let reg = /a{4,}/

let str = 'abcdddaaaabced'

console.log(str.search(reg))    // 6
```




#### 4. String.prototype.match(reg)
> 返回一个匹配结果具体信息的数组或者返回所有匹配到的结果

```
let str = 'abcdddaaaabced'

// 非全局匹配模式，只返回匹配到的第一个结果的具体信息
let reg1 = /dddaaa/
console.log(str.match(reg1)) // [ 'dddaaa', index: 3, input: 'abcdddaaaabced', groups: undefined ]


// 全局匹配
let reg2 = /a/g
let res = str.match(reg2)
console.log(res)   // [ 'a', 'a', 'a', 'a', 'a' ]

// 既返回整体匹配结果，也返回分组匹配到的结果
let date = '2021-01-12';

console.log(date.match(/(\d{4})-(\d{2})-(\d{2})/));

/*
[
  '2021-01-12',
  '2021',
  '01',
  '12',
  index: 0,
  input: '2021-01-12',
  groups: undefined
]
*/
```

#### 5. String.prototype.replace(substr｜RegExt , replaceStr)
> 将字符串某个子串或者满足正则表达式的字符串，替换成某个字符串
```

// 全部替换
let str = "$11, $22, $33";

let nStr = str.replace(/\$/g, '¥')

console.log(nStr)




// 第二个参数是callback函数的用法
let str = "abcdefabcdef";

let nStr = str.replace(/a(bc)/g, function(res, $1, resIndex, origin){
    // 第一个参数是整体匹配的结果，因为是全局替换，所以会查找两次，打印两次
    // 第二个参数是第一个分组匹配到的结果，bc
    // 第三个参数是整体匹配abc，找到的位置，第一次是0 ,第二次是6
    // 最后一个参数是要检索的源字符串
    return $1
})

console.log(nStr)


// 将连续出现的相同字符变成一个字符，例如将“aaaabbbbcccc11111aaaaa”变成"abc1a"
let str = "aaaabbbbcccc11111aaaaa";

let nStr = str.replace(/(.{1})\1*/g, function(res, $1){
    return $1
})

console.log(nStr)
```


#### 5. String.prototype.split(str/reg)
> 将字符串分割为数组，分隔符可以是字符串或者正则表达式
```
let str1 = 'abc'

let arr1 = str1.split('')

console.log(arr1)

let str2 = 'My name is tom'

let arr2 = str2.split(/\s/);
console.log(arr2)
```


#### 6. es6 y粘连修饰符，每次执行完正则表达式，指针回下移到字符串下一个字符，从头开始匹配。而不是像g一样，在剩余字符串整体查找找到就可以。 y可以用来匹配 连续 重复 的子串

```
let str = 'abc_abc_abc'

let reg1 = /abc/g

console.log(reg1.exec(str)) // [ 'abc', index: 0, input: 'abc_abc_abc', groups: undefined ]
console.log(reg1.exec(str)) // [ 'abc', index: 4, input: 'abc_abc_abc', groups: undefined ]

let reg2 = /abc/y 

console.log(reg2.exec(str)) // [ 'abc', index: 0, input: 'abc_abc_abc', groups: undefined ]
console.log(reg2.exec(str)) // [ 'abc', index: 4, input: 'abc_abc_abc', groups: undefined ]
```

#### 7. es6 dotAll增强，可以任意匹配

```
// 加上了u可以匹配unicode，加上了s，可以匹配换行符
let res = /aaa\nssss/us
```

#### 8. es6 命名分组捕获
> 在es5的match方法，匹配分组，可以拿到分组匹配结果，但是结果在一个数组内，用索引去取结果非常不方便
```
// 既返回整体匹配结果，也返回分组匹配到的结果
let date = '2021-01-12';

console.log(date.match(/(\d{4})-(\d{2})-(\d{2})/));

/*
[
  '2021-01-12',
  '2021',
  '01',
  '12',
  index: 0,
  input: '2021-01-12',
  groups: undefined
]
```

> 现在可以对匹配结果进行命名，然后通过 res.groupes[<name>]方式提取指定分组内容,特方便

```
let date = '2021-01-12';

let res = date.match(/(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/)

let groups = res.groups || {}

console.log(groups.year, groups.month, groups.day) // 2021 01 12
```



#### 9. es6 正则后行断言
> 先行断言，匹配某端字符串后面的 满足条件的字符串
```
let cont = 'hello world'

// 匹配hello后面的world，条件必须是hello后面的 world

cont.match(/hello(?=\sworld)/)
```

> 后行断言
```
// 匹配前面是hello，的后面的world

let res2 = cont.match(/(?<=hello\s)world/)  // ?<=前正则条件


console.log(res2)
```




### 练习
1. 将'$foo %foo foo'中, 前面是$的foo替换成bar
```
let res = cont.replace(/(?<=\$)foo/, 'bar');
console.log(res)
```
2. 请提取'$1 is worth about ¥7'中的美元数;
```
let cont = '$1 is worth about ¥7'

let res = cont.match(/(\$)(?<dollarNumber>\d*)/);

console.log(res.groups.dollarNumber)
```

3. 匹配字符串中所有的三位数 'abc124asad29832llasd109'
```
let cont = 'abc124asad29832llasd109'

let res = cont.match(/\d{3}/g);

console.log(res)
```


4. 将连续出现的相同字符变成一个字符，例如将“aaaabbbbcccc11111aaaaa”变成"abc1a"
```
let str = "aaaabbbbcccc11111aaaaa";

let nStr = str.replace(/(.{1})\1*/g, function(res, $1){
    return $1
})

console.log(nStr)
```


5. 通用的敏感字符过滤函数
```

// 敏感字符过滤替换
let content = '现在共厂党很好，草尼玛是脏话, 习景平是谁';
let dangerWords = ['共厂党', '草', '尼玛', '习景平']

function dangerWordsFilter(dangerWords, content, replacement) {
    let reg = new RegExp(`(${dangerWords.toString().replace(/,/g,'|')})`, 'g');
    return content.replace( reg, replacement)
}


let res = dangerWordsFilter(dangerWords, content, '*')
console.log(res)        // 现在*很好，**是脏话, *是谁

```