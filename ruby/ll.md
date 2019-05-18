# Ruby Test

姓名：
1. Ruby中，用来表示“相等”的方法有？


2. 以下表达式的返回值是？
```
->(o){ o*2 } === 2
```

3. Ruby中，实现一个计算年龄的方法，参数为生日。


3. Ruby / Rails中，如何定义类方法，尽可能列出你所知道的方法？


4. ES6 中，如何定义类方法？请代码示例。


5. Ruby中，如何定义实例变量，类变量，类实例变量？请代码示例。


6. Ruby中，定义一个方法`it`，可以这样被调用
```
 it('should work') { puts 'works!' }
```


7. 如何调用下面的 `total` 方法？

```
module Abc 
  class User
	  class << self 
      def total
        rand(10000) 
      end
    end 
  end
end
```



8. 写一段Ruby代码：把当前时间按照格式`YYYY-mm-dd H:M:S`以追加的方式写入文件`/tmp/date.log`中？



9. Rails应用中，有如下两个model，分别有定义如下字段，如何定义model间的关联关系，使其可以在ExpenseMember实例上调用 expense_items 方法？
```
class ExpenseMember < ActiveRecord::Base
  attribute :expense_id, :integer
  attribute :member_id, :integer 
  belongs_to :expense
  belongs_to :member
end

class ExpenseItem < ActiveRecord::Base
  attribute :expense_id, :integer
  attribute :member_id, :integer
  belongs_to :expense
  belongs_to :member
end

m = ExpenseMember.first
m.expense_items
```



10. 谈谈网站性能优化（200字以内）


