# 1 线索
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210808170624.png)


![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202205201818453.png)

# 2 定义
**数据操作与数据结构分离**。封装一些作用于某种数据结构中的**各元素的操作**，它可以在不改变这个数据结构的前提下定义作用于这些元素的新的操作。


# 3 场景
1. 对象**结构比较稳定**,但经常需要在此对象结构上定义新的操作。
2. 需要对一个对象结构中的对象进行很多不同的并且不相关的操作，而需要避免这些操作污染这些对象的类,也不希望在增加新操作时修改这些类。

# 4 应用
神策 SDK 中对发送数据的组装使用，数据每次组装不一样
```
{
	"lib":"Android",
	.......
}
```


# 5 结构
![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/202205201828856.png)

![](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210808170624.png)

抽象访问者：
	具体访问者
抽象元素：
	具体元素

# 6 优缺点
## 6.1 优点
1. 符合单一职责原则。访问者模式把相关的行为封装在一起，构成一个访问者，使每一个访问者的功能都比较单一。
2. 扩展性好。能够在不修改对象结构中的元素的情况下，为对象结构中的元素添加新的功能。
3. 复用性好。可以通过访问者来定义整个对象结构通用的功能，从而提高系统的复用程度。
4. 灵活性好。访问者模式将数据结构与作用于结构上的操作解耦，使得操作集合可相对自由地演化而不影响系统的数据结构。

## 6.2 缺点
1. 具体元素对访问者公布细节，违反了迪米特原则；破坏封装。访问者模式中具体元素对访问者公布细节，这破坏了对象的封装性。
2. 具体元素变更比较困难，增加新的元素类很困难。在访问者模式中，每增加一个新的元素类，都要在每一个具体访问者类中增加相应的具体操作，这违背了“开闭原则”。
3. 违反了依赖倒置原则，访问者模式依赖了具体类，而没有依赖抽象类。



# 7 举例-应用
1. Android中编译期注解(依赖APT(Annotation Processing Tools)实现), 其内部就有使用访问者模式，Element及其子类(包元素PackageElement，类型元素TypeElement等)是被访问者，其中的accept方法接收一个ElementVisitor类型的访问者，ElementVisitor中有多个visit方法处理不同类型的元素, 比较著名的ButterKnife,Dagger,Retrofit等开源库都有使用编译期注解实现
2. 通过 ASM 代码插桩。
3. JDK的 NIO模块下的Filevisitor。



# 8 访问者模式简介
访问者模式(Visitor Pattern)模式是**行为型(Behavioral)设计模式**，提供一个作用于某种对象结构上的各元素的操作方式，可以使我们在不改变元素结构的前提下，定义作用于元素的新操作。

换言之，如果系统的**数据结构是比较稳定的，但其操作（算法）是易于变化**的，那么使用访问者模式是个不错的选择；如果数据结构是易于变化的，则不适合使用访问者模式。

访问者模式一共有五种角色：

(1) Vistor（**抽象访问者**）：为该对象结构中具体元素角色声明一个访问操作接口。

(2) ConcreteVisitor（具体访问者）：每个具体访问者都实现了Vistor中定义的操作。

(3) Element（**抽象元素**）：定义了一个accept操作，以Visitor作为参数。

(4) ConcreteElement（具体元素）：实现了Element中的accept()方法，调用Vistor的访问方法以便完成对一个元素的操作。

(5) ObjectStructure（对象结构）：可以是组合模式，也可以是集合；能够枚举它包含的元素；提供一个接口，允许Vistor访问它的元素。

访问者模式类图如下:

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210801214141)


# 9 管理者-公司
## 9.1 抽象被访问者-Employee
公司员工抽象类  Employee.java
```java
/**
 * 公司员工（被访问者）抽象类
 *
 */
public abstract class Employee {
	
	/**
	 * 接收/引用一个抽象访问者对象
	 * @param department 抽象访问者 这里指的是公司部门如 人力资源部、财务部
	 */
	public abstract void accept(Department department);
 
}
```

## 9.2 具体被访问者
公司管理岗位员工类 ManagerEmployee.java
```java
/**
 * 公司员工：管理者（具体的被访问者对象）
 * 
 */
public class ManagerEmployee extends Employee {
	// 员工姓名
	private String name;
	// 每天上班时长
	private int timeSheet; 
	// 每月工资
	private double wage;
	// 请假/迟到 惩罚时长
	private int punishmentTime;
	
	public ManagerEmployee(String name, int timeSheet, double wage, int punishmentTime) {
		this.name = name;
		this.timeSheet = timeSheet;
		this.wage = wage;
		this.punishmentTime = punishmentTime;
	}
 
	
	@Override
	public void accept(Department department) {
		department.visit(this);
	}
	
	
	/**
	 * 获取每月的上班实际时长 = 每天上班时长 * 每月上班天数 - 惩罚时长
	 * @return
	 */
	public int getTotalTimeSheet(){
		return timeSheet * 22 - punishmentTime;
	}
	
	
	/**
	 * 获取每月实际应发工资 = 每月固定工资 - 惩罚时长 * 5<br/>
	 * <作为公司管理者 每迟到1小时 扣5块钱>
	 * @return
	 */
	public double getTotalWage(){
		return wage - punishmentTime * 5;
	}
	
	public String getName() {
		return name;
	}
 
	public void setName(String name) {
		this.name = name;
	}
 
	public double getWage() {
		return wage;
	}
 
	public void setWage(double wage) {
		this.wage = wage;
	}
	
	public int getPunishmentTime() {
		return punishmentTime;
	}
 
	public void setPunishmentTime(int punishmentTime) {
		this.punishmentTime = punishmentTime;
	}
	
}

```

公司普通岗位员工类 GeneralEmployee.java
```java
/**
 * 公司普通员工（具体的被访问者对象）
 *
 */
public class GeneralEmployee extends Employee {
    // 员工姓名
	private String name;
	// 每天上班时长
	private int timeSheet;
	// 每月工资
	private double wage;
	// 请假/迟到 惩罚时长
	private int punishmentTime;
 
	public GeneralEmployee(String name, int timeSheet, double wage, int punishmentTime) {
		this.name = name;
		this.timeSheet = timeSheet;
		this.wage = wage;
		this.punishmentTime = punishmentTime;
	}
 
	@Override
	public void accept(Department department) {
		department.visit(this);
	}
 
	/**
	 * 获取每月的上班实际时长 = 每天上班时长 * 每月上班天数 - 惩罚时长
	 * @return
	 */
	public int getTotalTimeSheet() {
		return timeSheet * 22 - punishmentTime;
	}
 
	/**
	 * 获取每月实际应发工资 = 每月固定工资 - 惩罚时长 * 10<br/>
	 * <作为公司普通员工  每迟到1小时 扣10块钱  坑吧？  哈哈>
	 * 
	 * @return
	 */
	public double getTotalWage() {
		return wage - punishmentTime * 10;
	}
	
	
	public String getName() {
		return name;
	}
 
	public void setName(String name) {
		this.name = name;
	}
 
	public double getWage() {
		return wage;
	}
 
	public void setWage(double wage) {
		this.wage = wage;
	}
 
	public int getPunishmentTime() {
		return punishmentTime;
	}
 
	public void setPunishmentTime(int punishmentTime) {
		this.punishmentTime = punishmentTime;
	}
 
}

```

如果要更改为人力资源部对员工的一个月的上班时长统计 则只要将上述代码中的

```java
FADepartment department = new FADepartment();
```

修改为如下即可

```java
HRDepartment department = new HRDepartment();
```

8.程序运行结果：

```java
管理者: 王总  固定工资 =20000.0, 迟到时长 10小时, 实发工资=19950.0管理者: 谢经理  固定工资 =15000.0, 迟到时长 15小时, 实发工资=14925.0普通员工: 小杰  固定工资 =8000.0, 迟到时长 8小时, 实发工资=7920.0普通员工: 小晓  固定工资 =8500.0, 迟到时长 12小时, 实发工资=8380.0普通员工: 小虎  固定工资 =7500.0, 迟到时长 0小时, 实发工资=7500.0
```

## 9.3 抽象访问者
公司部门抽象类 Department.java
```java
/**
 * 公司部门（访问者）抽象类
 *
 */
public abstract class Department {
	
	// 声明一组重载的访问方法，用于访问不同类型的具体元素（这里指的是不同的员工）  
	
	/**
	 * 抽象方法 访问公司管理者对象<br/>
	 * 具体访问对象的什么  就由具体的访问者子类（这里指的是不同的具体部门）去实现
	 * @param me
	 */
	public abstract void visit(ManagerEmployee me);
	
	/**
	 * 抽象方法 访问公司普通员工对象<br/>
	 * 具体访问对象的什么  就由具体的访问者子类（这里指的是不同的具体部门）去实现
	 * @param ge
	 */
	public abstract void visit(GeneralEmployee ge);
 
}

```

## 9.4 具体访问者
公司财务部类 FADepartment.java
```java
/**
 * 具体访问者对象：公司财务部<br/>
 * 财务部的职责就是负责统计核算员工的工资
 *
 */
public class FADepartment extends Department {
 
	/**
	 * 访问公司管理者对象的每月工资
	 */
	@Override
	public void visit(ManagerEmployee me) {
		double totalWage = me.getTotalWage();
		System.out.println("管理者: " + me.getName() + 
				"  固定工资 =" + me.getWage() + 
				", 迟到时长 " + me.getPunishmentTime() + "小时"+
				", 实发工资="+totalWage);
	}
 
	/**
	 * 访问公司普通员工对象的每月工资
	 */
	@Override
	public void visit(GeneralEmployee ge) {
		double totalWage = ge.getTotalWage();
		System.out.println("普通员工: " + ge.getName() + 
				"  固定工资 =" + ge.getWage() + 
				", 迟到时长 " + ge.getPunishmentTime() + "小时"+
				", 实发工资="+totalWage);
	}
 
}

```

具体访问者：公司人力资源部类 HRDepartment.java
```java
/**
 * 具体访问者对象：公司人力资源部<br/>
 * 人力资源部的职责就是负责统计核算员工的每月上班时长
 * @author  lvzb.software@qq.com
 *
 */
public class HRDepartment extends Department {
 
	/**
	 * 访问公司管理者对象的每月实际上班时长统计
	 */
	@Override
	public void visit(ManagerEmployee me) {
		me.getTotalTimeSheet();
	}
 
	/**
	 * 访问公司普通员工对象的每月实际上班时长统计
	 */
	@Override
	public void visit(GeneralEmployee ge) {
		ge.getTotalTimeSheet();
	}
 
}

```

## 9.5 客户端测试类
模拟财务部对公司员工的工资核算和访问 Client.java
```
import java.util.ArrayList;
import java.util.List;
 
public class Client {
 
	public static void main(String[] args) {
		List<Employee> employeeList = new ArrayList<Employee>();
		Employee mep1,mep2,gep1,gep2,gep3;
		// 管理者1
		mep1 = new ManagerEmployee("王总", 8, 20000, 10);
		// 管理者2
		mep2 = new ManagerEmployee("谢经理", 8, 15000, 15);
		// 普通员工1
		gep1 = new GeneralEmployee("小杰", 8, 8000, 8);
		// 普通员工2
		gep2 = new GeneralEmployee("小晓", 8, 8500, 12);
		// 普通员工3
		gep3 = new GeneralEmployee("小虎", 8, 7500, 0);
		
		employeeList.add(mep1);
		employeeList.add(mep2);
		employeeList.add(gep1);
		employeeList.add(gep2);
		employeeList.add(gep3);
		
		// 财务部 对公司员工的工资核算/访问
		FADepartment department = new FADepartment();
		for(Employee employee : employeeList){
			employee.accept(department);
		}	
	}
	
}

```

# 10 访问者模式举例

如果老师教学反馈得分大于等于85分、学生成绩大于等于90分，则可以入选成绩优秀奖；如果老师论文数目大于8、学生论文数目大于2，则可以入选科研优秀奖。

在这个例子中，老师和学生就是Element，他们的数据结构稳定不变。从上面的描述中，我们发现，对数据结构的操作是多变的，一会儿评选成绩，一会儿评选科研，这样就适合使用**访问者模式来分离数据结构和操作**。

| 序号 | 类名                | 角色            | 说明       |
| ---- | ------------------- | --------------- | ---------- |
| 1    | Visitor             | Visitor         | 抽象访问者 |
| 2    | GradeSelection      | ConcreteVisitor | 具体访问者 |
| 3    | ResearcherSelection | ConcreteVisitor | 具体访问者 |
| 4    | Element             | Element         | 抽象元素   |
| 5    | Teacher             | ConcreteElement | 具体元素   |
| 6    | Student             | ConcreteElement | 具体元素   |
| 7    | ObjectStructure     | ObjectStructure | 对象结构   |
| 8    | VisitorClient       | 客户端          | 演示调用   |

举例的类图如下：

![img](http://wupan.dns.army:5000/wupan/Typora-Picgo-Gitee/raw/branch/master/img/20210801214218)


## 10.1 Visitor 抽象访问者
```java
/**
 * 抽象访问者，为该对象结构中具体元素角色声明一个访问操作接口。
 */
public interface Visitor {

    void visit(Student element);

    void visit(Teacher element);

}
```

## 10.2 GradeSelection 选拔优秀成绩者
```java
/**
 * 具体访问者，实现了Vistor中定义的操作。
 */
public class GradeSelection implements Visitor {

    private String awardWords = "[%s]的分数是%d，荣获了成绩优秀奖。";

    @Override
    public void visit(Student element) {
        // 如果学生考试成绩超过90，则入围成绩优秀奖。
        if (element.getGrade() >= 90) {
            System.out.println(String.format(awardWords, 
                    element.getName(), element.getGrade()));
        }
    }

    @Override
    public void visit(Teacher element) {
        // 如果老师反馈得分超过85，则入围成绩优秀奖。
        if (element.getScore() >= 85) {
            System.out.println(String.format(awardWords, 
                    element.getName(), element.getScore()));
        }
    }
}
```

## 10.3 ResearcherSelection，选拔优秀科研者
```java
/**
 * 具体访问者，实现了Vistor中定义的操作。
 */
public class ResearcherSelection implements Visitor {

    private String awardWords = "[%s]的论文数是%d，荣获了科研优秀奖。";

    @Override
    public void visit(Student element) {
        // 如果学生发表论文数超过2，则入围科研优秀奖。
        if(element.getPaperCount() > 2){
            System.out.println(String.format(awardWords,
                    element.getName(),element.getPaperCount()));
        }
    }

    @Override
    public void visit(Teacher element) {
        // 如果老师发表论文数超过8，则入围科研优秀奖。
        if(element.getPaperCount() > 8){
            System.out.println(String.format(awardWords,
                    element.getName(),element.getPaperCount()));
        }
    }
}
```

## 10.4 Element，抽象元素角色
```java
/**
 * 抽象元素角色，定义了一个accept操作，以Visitor作为参数。
 */
public interface Element {

    //接受一个抽象访问者访问
    void accept(Visitor visitor);

}
```

## 10.5 5.Teacher，具体元素
```java
/**
 * 具体元素，允许visitor访问本对象的数据结构。
 */
public class Teacher implements Element {

    private String name; // 教师姓名
    private int score; // 评价分数
    private int paperCount; // 论文数

    // 构造器
    public Teacher(String name, int score, int paperCount) {
        this.name = name;
        this.score = score;
        this.paperCount = paperCount;
    }

    // visitor访问本对象的数据结构
    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getScore() {
        return score;
    }

    public void setScore(int score) {
        this.score = score;
    }

    public int getPaperCount() {
        return paperCount;
    }

    public void setPaperCount(int paperCount) {
        this.paperCount = paperCount;
    }
}
```

## 10.6 Student，具体元素
```java
/**
 * 具体元素，允许visitor访问本对象的数据结构。
 */
public class Student implements Element {

    private String name; // 学生姓名
    private int grade; // 成绩
    private int paperCount; // 论文数

    // 构造器
    public Student(String name, int grade, int paperCount) {
        this.name = name;
        this.grade = grade;
        this.paperCount = paperCount;
    }

    // visitor访问本对象的数据结构
    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getGrade() {
        return grade;
    }

    public void setGrade(int grade) {
        this.grade = grade;
    }

    public int getPaperCount() {
        return paperCount;
    }

    public void setPaperCount(int paperCount) {
        this.paperCount = paperCount;
    }
}
```

## 10.7 7.ObjectStructure， 对象结构
```java
/**
 * 对象结构，是元素的集合，提供元素的访问入口。
 */
public class ObjectStructure {

    // 使用集合保存Element元素，示例没有考虑多线程的问题。
    private ArrayList<Element> elements = new ArrayList<>();

    /**
     * 访问者访问元素的入口
     *
     * @param visitor 访问者
     */
    public void accept(Visitor visitor) {
        for (int i = 0; i < elements.size(); i++) {
            Element element = elements.get(i);
            element.accept(visitor);
        }
    }

    /**
     * 把元素加入到集合
     *
     * @param element 待添加的元素
     */
    public void addElement(Element element) {
        elements.add(element);
    }

    /**
     * 把元素从集合中移除
     *
     * @param element 要移除的元素
     */
    public void removeElement(Element element) {
        elements.remove(element);
    }
}
```

## 10.8 8.VisitorClient 客户端
```java
/**
 * 如果教师发表论文数超过8篇或者学生论文超过2篇可以评选科研优秀奖，
 * 如果教师教学反馈分大于等于85分或者学生成绩大于等于90分可以评选成绩优秀奖。
 */
public class VisitorClient {

    public static void main(String[] args) {
        // 初始化元素
        Element stu1 = new Student("Student Jim", 92, 3);
        Element stu2 = new Student("Student Ana", 89, 1);
        Element t1 = new Teacher("Teacher Mike", 83, 10);
        Element t2 = new Teacher("Teacher Lee", 88, 7);
        // 初始化对象结构
        ObjectStructure objectStructure = new ObjectStructure();
        objectStructure.addElement(stu1);
        objectStructure.addElement(stu2);
        objectStructure.addElement(t1);
        objectStructure.addElement(t2);
        // 定义具体访问者，选拔成绩优秀者
        Visitor gradeSelection = new GradeSelection();
        // 具体的访问操作，打印输出访问结果
        objectStructure.accept(gradeSelection);
        System.out.println("----结构不变，操作易变----");
        // 数据结构是没有变化的，如果我们还想增加选拔科研优秀者的操作，那么如下。
        Visitor researcherSelection = new ResearcherSelection();
        objectStructure.accept(researcherSelection);
    }
}
```

结果输出
```java
[Student Jim]的分数是92，荣获了成绩优秀奖。
[Teacher Lee]的分数是88，荣获了成绩优秀奖。
----结构不变，操作易变----
[Student Jim]的论文数是3，荣获了科研优秀奖。
[Teacher Mike]的论文数是10，荣获了科研优秀奖。
```


# 11 总结
如果一个对象结构比较复杂，同时结构稳定不易变化，但却需要经常在此结构上定义新的操作，那就非常合适使用访问者模式，比如复杂的集合对象、XML文档解析、编译器的设计等。

访问者模式使我们更加容易的增加访问操作，但增加元素比较困难，需要我们修改抽象访问类和所有的具体访问类。

# 12 参考

https://www.jianshu.com/p/cd17bae4e949
