---
layout: post
title:  "基于Android热更新Robust系统的Javassist和ASM对比"
date:   2017-06-01 15:26:12 +0800
categories: Robust系列
---
#简述
最初在Android热更新项目[Robust](https://github.com/Meituan-Dianping/Robust)使用Javassist工具，为了项目的快速上线，Javassist的学习成本比较低，直接插入的是Java代码，不用去理会底层的各种字节码指令和栈帧结构，使得Robust项目在初期进展很快，但是随着[Robust](https://github.com/Meituan-Dianping/Robust)的快速推广，使用Javassist的问题就暴露出来；但是ASM就是一个更好的选择吗？其实也不一定，因为ASM这个工具固然优秀，但是学习成本比较，需要对底层的自己字节码指令有一些多多少少的了解，这就增加了学习的难度，导致并不容易快速上手。  

在这篇文章，我们并不讨论Javassist和ASM之间的性能之类的区别，仅仅从一个项目的角度来查看到底哪一个工具更加适合，两者各有千秋，“存在即是合理的”，当然类似的字节码工具还有很多，受限于各大公司的技术栈和场景，大家选取的工具可能略有不同。
#插桩目标
我们先介绍一下我们的插桩目标（以[Robust项目](https://github.com/Meituan-Dianping/Robust)为例）,原始函数如下：

```java
	public long getIndex() {
        return 100;
    }
```
我们插桩完成的目标是：

```java
	public static ChangeQuickRedirect changeQuickRedirect;
    public long getIndex() {
            //PatchProxy中封装了获取当前className和methodName的逻辑，并在其内部最终调用了changeQuickRedirect的对应函数
            if(PatchProxy.isSupport(new Object[0], this, changeQuickRedirect, false,1,new Class[]{},long.class)) {
                return ((Long)PatchProxy.accessDispatch(new Object[0], this, changeQuickRedirect, false,1,new Class[]{},long.class)).longValue();
            }
         
        return 100L;
    }
```
就是在每个函数前面插入一段代码，让我们来看一下ASM和Javassist的分别实现起来的难度。

# Javassist的使用
使用Javassist来进行字节码操作就是一个字：爽。由于写的也是Java代码，极其容易上手，想用尝试字节码工具的软件工程师来说，这个无疑是最好的一个选择。先让我们看看[Robust](https://github.com/Meituan-Dianping/Robust)是如何使用Javassist插入代码,下述代码位于Robust项目类```JavaAssistInsertImpl```中的方法```insertCode(List<CtClass> box, File jarFile) throws CannotCompileException, IOException, NotFoundException```的部分代码片段：

```java
	boolean isStatic = (ctMethod.getModifiers() & AccessFlag.STATIC)!= 0;
    CtClass returnType = ctMethod.getReturnType();
    String returnTypeString = returnType.getName();
    String body = "Object argThis = null;";
    if (!isStatic) {
         body += "argThis = $0;";
    }
    String parametersClassType=getParametersClassType(ctMethod);
    body += "   if (com.meituan.robust.PatchProxy.isSupport($args, argThis, "+Constants.INSERT_FIELD_NAME+", "+isStatic+
            ", " + methodMap.get(ctBehavior.getLongName()) + ","+parametersClassType+","+returnTypeString+".class)) {";
    body += getReturnStatement(returnTypeString, isStatic, methodMap.get(ctBehavior.getLongName()),parametersClassType,returnTypeString+".class");
    body += "   }";
    
   
    //这才是真正插桩函数，只有一句话！！！
    ctBehavior.insertBefore(body);

```
一直截止到上述代码的最后一句，我们是在根据上下文来拼接插入的参数，最后一句是插桩代码，只有一句话，上述拼接的参数全是Java代码，只要熟悉基本的Javassist的API就可以进行开发，是不是有点小激动
或许你觉得上。```ctBehavior.insertBefore```这个函数的参数就是插入代码的字符串，你要做的就是根据上下文的环境拼接你插入的代码，Javassist的使用手册，可以参看[官方的文档](http://jboss-javassist.github.io/javassist/),具体的[API手册](http://jboss-javassist.github.io/javassist/tutorial/tutorial.html)。

#ASM的插桩使用
上面介绍完了Javassist的插桩使用，下面我们来看看ASM要实现上面的插桩需要做哪些事情,下述代码位于Robust项目类```RobustAsmUtils```中的函数```createInsertCode(GeneratorAdapter mv, String className, List<Type> args, Type returnType, boolean isStatic, int methodId)```，

```java
/**
		 * 调用isSupport方法
		 */
		prepareMethodParameters(mv,className,args,returnType,isStatic,methodId);
		//开始调用
		mv.visitMethodInsn(Opcodes.INVOKESTATIC,
				PROXYCLASSNAME,
				"isSupport",
				"([Ljava/lang/Object;Ljava/lang/Object;"+REDIRECTCLASSNAME+"ZI[Ljava/lang/Class;Ljava/lang/Class;)Z");
		Label l1 = new Label();
		mv.visitJumpInsn(Opcodes.IFEQ, l1);
		prepareMethodParameters(mv,className,args,returnType,isStatic,methodId);
		//开始调用
		mv.visitMethodInsn(Opcodes.INVOKESTATIC,
				PROXYCLASSNAME,
				"accessDispatch",
				"([Ljava/lang/Object;Ljava/lang/Object;"+REDIRECTCLASSNAME+"ZI[Ljava/lang/Class;Ljava/lang/Class;)Ljava/lang/Object;");

		//判断是否有返回值，代码不同
		if("V".equals(returnType.getDescriptor())){
			mv.visitInsn(Opcodes.POP);
			mv.visitInsn(Opcodes.RETURN);
		}else{
			//强制转化类型
			if(!castPrimateToObj(mv, returnType.getDescriptor())){
				//这里需要注意，如果是数组类型的直接使用即可，如果非数组类型，就得去除前缀了,还有最终是没有结束符;
				//比如：Ljava/lang/String; ==》 java/lang/String
				String newTypeStr = null;
				int len = returnType.getDescriptor().length();
				if(returnType.getDescriptor().startsWith("[")){
					newTypeStr = returnType.getDescriptor().substring(0, len);
				}else{
					newTypeStr = returnType.getDescriptor().substring(1, len-1);
				}
				mv.visitTypeInsn(Opcodes.CHECKCAST, newTypeStr);
			}

			//这里还需要做返回类型不同返回指令也不同
			mv.visitInsn(getReturnTypeCode(returnType.getDescriptor()));
		}

		mv.visitLabel(l1);
``` 
上述只是代码片段，整个ASM的插桩代码还是比较庞大的，就不一一列出了，有兴趣的可以参看```RobustAsmUtils```。

对了一下ASM和Javassist的插桩代码，是不是一目了然，两者的代码量和代码的复杂程度都不是一个量级的，而且ASM的代码涉及到大量函数调用，给我的最直观的感觉就是，写ASM的代码就是类似于编译原理的四元表达式，类似于在写汇编的感觉，编码难度很大。

虽然官方有提供一个工具[ASMifier](http://asm.ow2.org/doc/faq.html#Q10)来生成代码，但是明显感觉到文档更新不及时，最新的ASM5.0.3按照文档的步骤无法成功生成字节码代码，也是比较心累的一项，需要自行找[解决办法](http://yangbolin.cn/2014/07/27/how-to-use-asmifier/)。

#总结
Javassist的字节码工具在在处理其他字节码工具生成的代码时出现了bug，表现最为突出的就是AspectJ，当我们使用AspectJ的时候，AspectJ会生成一些类, Javassist处理这些类的特定情况下才会出现问题，一般不会遇到。所以你的项目如果比较简单，没有使用众多不同的字节码工具，最优的方法肯定是Javassist啦，ASM学习和使用成本有点高，虽然功能很强大。

