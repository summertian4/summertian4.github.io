title: ID3算法详解
date: 2015-05-28 16:40
tags:
  - bigData
categories:
  - 大数据
---


这学期hadoop的作业是实现ID3算法，在网上找到了一篇非常好的资料，但是代码没有详细介绍。研究一番之后写出了自己ID3算法的普通实现和线程模拟分布式实现。

先附上原文地址：
http://www.cnblogs.com/zhangchaoyang/articles/2196631.html
博主写的非常好，基本上简单易懂的描述了ID3算法的原理。

## 问题
统计了14天的气象数据(指标包括outlook，temperature，humidity，windy)，并已知这些天气是否打球(play)。如果给出新一天的气象指标数据:sunny,cool,high,TRUE，判断一下会不会去打球。
## ID3原理
直接看博客链接，非常详细http://www.cnblogs.com/zhangchaoyang/articles/2196631.html
## 代码实现
别忘了放入输入文件，按我的程序保存在指定位置，保存名为weather.arff

<!--more-->

```
@relation weather.symbolic
 
@attribute outlook {sunny, overcast, rainy}
@attribute temperature {hot, mild, cool}
@attribute humidity {high, normal}
@attribute windy {TRUE, FALSE}
@attribute play {yes, no}
 
@data
sunny,hot,high,FALSE,no
sunny,hot,high,TRUE,no
overcast,hot,high,FALSE,yes
rainy,mild,high,FALSE,yes
rainy,cool,normal,FALSE,yes
rainy,cool,normal,TRUE,no
overcast,cool,normal,TRUE,yes
sunny,mild,high,FALSE,no
sunny,cool,normal,FALSE,yes
rainy,mild,normal,FALSE,yes
sunny,mild,normal,TRUE,yes
overcast,mild,high,TRUE,yes
overcast,hot,normal,FALSE,yes
rainy,mild,high,TRUE,no
```

```
package com.coderfish.id3;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileReader;
import java.io.FileWriter;
import java.io.IOException;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.LinkedList;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import org.dom4j.Document;
import org.dom4j.DocumentHelper;
import org.dom4j.Element;
import org.dom4j.io.OutputFormat;
import org.dom4j.io.XMLWriter;

/**
 * date:2015年6月15日<br/ >
 * description:
 * 统计了14天的气象数据(outlook天气状况，temperature温度，humidity湿度，windy风力)，并已知这些天气是否打球(play)
 * 如果给出新一天的气象指标数据:sunny,cool,high,TRUE，判断一下会不会去打球。<br/ >
 * 
 * @author 周凌宇 <br/ >
 * 
 */
public class ID3 {
	/**
	 * 存储属性的名称
	 */
	private ArrayList<String> attribute = new ArrayList<String>();
	/**
	 * 存储每个属性的取值
	 */
	private ArrayList<ArrayList<String>> attributeValue = new ArrayList<ArrayList<String>>();
	/**
	 * 原始数据
	 */
	private ArrayList<String[]> data = new ArrayList<String[]>();
	/**
	 * 决策变量在属性集中的索引
	 */
	int decisionIndex;
	/**
	 * ARFF格式
	 */
	public static final String patternString = "@attribute(.*)[{](.*?)[}]";
	/**
	 * XML文件对象
	 */
	Document xmldoc;
	/**
	 * 根节点
	 */
	Element root;

	/**
	 * 构造方法<br/ >
	 * 新建xml<br/ >
	 * <root><br/ >
	 * <DecisionTree value="null"><br/ >
	 * </DecisionTree><br/ >
	 * </root>
	 */
	public ID3() {
		xmldoc = DocumentHelper.createDocument();
		root = xmldoc.addElement("root");
		root.addElement("DecisionTree").addAttribute("value", "null");
	}

	public static void main(String[] args) {
		ID3 inst = new ID3();
		inst.readARFF(new File("f:\\id3temp\\weather.arff"));
		inst.setDec("play");
		// conditionList存放条件个数
		LinkedList<Integer> conditionNumList = new LinkedList<Integer>();
		for (int i = 0; i < inst.attribute.size(); i++) {
			// 如果i！=决策变量索引（实际5）
			if (i != inst.decisionIndex)
				conditionNumList.add(i);
		}
		// dataList存放data的个数
		ArrayList<Integer> dataNumList = new ArrayList<Integer>();
		for (int i = 0; i < inst.data.size(); i++) {
			dataNumList.add(i);
		}
		inst.buildDT("DecisionTree", "null", dataNumList, conditionNumList);
		inst.writeXML("f:\\id3temp\\dt.xml");
		return;
	}

	/**
	 * 读取arff文件，给attribute、attributevalue、data赋值
	 * 
	 * @param file
	 *            要读取的文件
	 */
	public void readARFF(File file) {
		try {
			FileReader fr = new FileReader(file);
			BufferedReader br = new BufferedReader(fr);
			String line;
			// 设置arff模式
			Pattern pattern = Pattern.compile(patternString);
			// 读取数据
			while ((line = br.readLine()) != null) {
				// 判断合规
				Matcher matcher = pattern.matcher(line);
				// 尝试查找与该模式匹配的输入序列的下一个子序列
				if (matcher.find()) {
					// 返回匹配到的子字符串
					attribute.add(matcher.group(1).trim());
					String[] values = matcher.group(2).split(",");
					ArrayList<String> al = new ArrayList<String>(values.length);
					for (String value : values) {
						al.add(value.trim());
					}
					// 存入attributevalue对象中
					attributeValue.add(al);
				}
				// 判断开头
				else if (line.startsWith("@data")) {
					while ((line = br.readLine()) != null) {
						if (line == "")
							continue;
						String[] row = line.split(",");
						data.add(row);
					}
				} else {
					continue;
				}
			}
			br.close();
		} catch (IOException e1) {
			e1.printStackTrace();
		}
	}

	/**
	 * 设置决策变量索引
	 * 
	 * @param n
	 */
	public void setDec(int n) {
		if (n < 0 || n >= attribute.size()) {
			System.err.println("决策变量指定错误。");
			System.exit(2);
		}
		decisionIndex = n;
	}

	/**
	 * 设置决策变量
	 * 
	 * @param name
	 */
	public void setDec(String name) {
		// 获取下标
		int n = attribute.indexOf(name);
		setDec(n);
	}

	/**
	 * 计算熵
	 * 
	 * @param arr
	 *            yes or no个数
	 * @return 熵
	 */
	public double getEntropy(int[] arr) {
		double entropy = 0.0;
		int sum = 0;
		for (int i = 0; i < arr.length; i++) {
			entropy -= arr[i] * Math.log(arr[i] + Double.MIN_VALUE)
					/ Math.log(2);
			sum += arr[i];
		}
		// log(N)/log(2)
		entropy += sum * Math.log(sum + Double.MIN_VALUE) / Math.log(2);
		entropy /= sum;
		return entropy;
	}

	/**
	 * 判断节点值是否全部相同
	 * 
	 * @param subset
	 *            给定子集
	 * @return 全部相同返回true，有不相同返回false
	 */
	public boolean infoPure(ArrayList<Integer> subset) {
		// yes or no
		String value = data.get(subset.get(0))[decisionIndex];
		for (int i = 1; i < subset.size(); i++) {
			String next = data.get(subset.get(i))[decisionIndex];
			// 如果该结点值和第一个节点值不同
			if (!value.equals(next))
				return false;
		}
		return true;
	}

	/**
	 * 给定原始数据的子集(subset中存储行号),当以第index个属性为节点时计算它的信息熵
	 * 
	 * @param subset
	 *            给定子集
	 * @param index
	 *            属性下标
	 * @return
	 */
	public double calNodeEntropy(ArrayList<Integer> subset, int index) {
		int sum = subset.size();
		double entropy = 0.0;
		// 长度为属性个数 存放每种可能的决策情况
		int[][] info = new int[attributeValue.get(index).size()][];
		for (int i = 0; i < info.length; i++)
			// 决策个数
			info[i] = new int[attributeValue.get(decisionIndex).size()];
		// 长度为属性个数
		int[] count = new int[attributeValue.get(index).size()];
		for (int i = 0; i < sum; i++) {
			// 子集遍历下标
			int n = subset.get(i);
			String nodeValue = data.get(n)[index];
			int nodeIndex = attributeValue.get(index).indexOf(nodeValue);
			count[nodeIndex]++;
			// 决策结果
			String decValue = data.get(n)[decisionIndex];
			int decIndex = attributeValue.get(decisionIndex).indexOf(decValue);
			info[nodeIndex][decIndex]++;
		}
		for (int i = 0; i < info.length; i++) {
			// 加权求值
			entropy += getEntropy(info[i]) * count[i] / sum;
		}
		return entropy;
	}

	/**
	 * 构建决策树 第一次给定值为"DecisionTree", "null", al, ll
	 * 
	 * @param name
	 *            节点名称
	 * @param value
	 *            节点值
	 * @param subset
	 *            子集
	 * @param conditionList
	 */
	public void buildDT(String name, String value,
			ArrayList<Integer> dataNumList, LinkedList<Integer> conditionNumList) {
		Element ele = null;
		// 从任意位置的节点上选择名称为 name 的节点。
		List<Element> list = root.selectNodes("//" + name);
		Iterator<Element> iter = list.iterator();
		while (iter.hasNext()) {
			ele = iter.next();
			// 节点匹配value跳出循环
			if (ele.attributeValue("value").equals(value))
				break;
		}
		// 如果子决策全部相同 直接放入决策 跳出递归
		if (infoPure(dataNumList)) {
			ele.setText(data.get(dataNumList.get(0))[decisionIndex]);
			return;
		}
		int minIndex = -1;
		double minEntropy = Double.MAX_VALUE;
		for (int i = 0; i < conditionNumList.size(); i++) {
			if (i == decisionIndex)
				continue;
			// 给定子集计算熵
			double entropy = calNodeEntropy(dataNumList,
					conditionNumList.get(i));
			if (entropy < minEntropy) {
				minIndex = conditionNumList.get(i);
				minEntropy = entropy;
			}
		}
		// 找出最小熵的节点属性名（作为要加入节点名）
		String nodeName = attribute.get(minIndex);
		conditionNumList.remove(new Integer(minIndex));
		// 拿出对应属性的所有值
		ArrayList<String> attvalues = attributeValue.get(minIndex);
		for (String val : attvalues) {
			ele.addElement(nodeName).addAttribute("value", val);
			ArrayList<Integer> al = new ArrayList<Integer>();
			for (int i = 0; i < dataNumList.size(); i++) {
				// 找出开头为val的数据行
				if (data.get(dataNumList.get(i))[minIndex].equals(val)) {
					al.add(dataNumList.get(i));
				}
			}
			buildDT(nodeName, val, al, conditionNumList);
		}
	}

	/**
	 * xml输出
	 * 
	 * @param filename
	 */
	public void writeXML(String filename) {
		try {
			File file = new File(filename);
			if (!file.exists())
				file.createNewFile();
			FileWriter fw = new FileWriter(file);
			OutputFormat format = OutputFormat.createPrettyPrint(); // 美化格式
			XMLWriter output = new XMLWriter(fw, format);
			output.write(xmldoc);
			output.close();
		} catch (IOException e) {
			System.out.println(e.getMessage());
		}
	}
}
```

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


