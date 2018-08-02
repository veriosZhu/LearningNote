# Activity工作流程


## 目录


## 流程图
```flow
st=>start: Start|past:>http://blog.xiaoyulive.top
e=>end: End:>http://www.xiaoyulive.top
op1=>operation: My Operation|past
op2=>operation: Stuff|current
sub1=>subroutine: My Subroutine|invalid
cond=>condition: Yes or No?|approved:>https://github.com/quanzaiyu
c2=>condition: Good idea|rejected
io=>inputoutput: catch something...|request

st->op1(right)->cond
cond(yes, right)->c2
cond(no)->sub1(left)->op1
c2(yes)->io->e
c2(no)->op2->e
```
