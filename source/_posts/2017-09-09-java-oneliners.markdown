---
layout: post
title: "[每周一译]Java中的十个\"单行代码编程\"(One Liner)"
date: 2017-09-09 19:29:34 +0800
comments: true
categories: java translate dzone
---

本文列举了十个使用一行代码即可独立完成(不依赖其他代码)的业务逻辑，主要依赖的是Java8中的Lambda和Stream等新特性以及try-with-resources、JAXB等。

<!--more-->

1. 对列表/数组中的每个元素都乘以2

	```
	// Range是半开区间
	int[] ia = range(1, 10).map(i -> i * 2).toArray();
	List<Integer> result = range(1, 10).map(i -> i * 2).boxed().collect(toList());
   ```
    
2. 计算集合/数组中的数字之和

	```
	range(1, 1000).sum();
  	range(1, 1000).reduce(0, Integer::sum);
   	Stream.iterate(0, i -> i + 1).limit(1000).reduce(0, Integer::sum);
   	IntStream.iterate(0, i -> i + 1).limit(1000).reduce(0, Integer::sum);
   ```

3. 验证字符串是否包含集合中的某一字符串

	```
   final List<String> keywords = Arrays.asList("brown", "fox", "dog", "pangram");
   final String tweet = "The quick brown fox jumps over a lazy dog. #pangram http://www.rinkworks.com/words/pangrams.shtml";

   keywords.stream().anyMatch(tweet::contains);
   keywords.stream().reduce(false, (b, keyword) -> b || tweet.contains(keyword), (l, r) -> l || r);
   ```
    
4. 读取文件内容

	> 原作者认为try with resources也是一种单行代码编程。
	
	```
	try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
	  String fileText = reader.lines().reduce("", String::concat);
	}
	
	try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
	  List<String> fileLines = reader.lines().collect(toCollection(LinkedList<String>::new));
	}
	
	try (Stream<String> lines = Files.lines(new File("data.txt").toPath(), Charset.defaultCharset())) {
	  List<String> fileLines = lines.collect(toCollection(LinkedList<String>::new));
	}
    ```
    
5. 输出歌曲《Happy Birthday to You!》 - 根据集合中不同的元素输出不同的字符串

	```
   	range(1, 5).boxed().map(i -> { out.print("Happy Birthday "); if (i == 3) return "dear NAME"; else return "to You"; }).forEach(out::println);
	```

6. 过滤并分组集合中的数字

	```
    Map<String, List<Integer>> result = Stream.of(49, 58, 76, 82, 88, 90).collect(groupingBy(forPredicate(i -> i > 60, "passed", "failed")));
	```
    
7. 获取并解析xml协议的Web Service

	```
   FeedType feed = JAXB.unmarshal(new URL("http://search.twitter.com/search.atom?&q=java8"), FeedType.class);
   JAXB.marshal(feed, System.out);
   ```
    
8. 获得集合中最小/最大的数字

	```
	int min = Stream.of(14, 35, -7, 46, 98).reduce(Integer::min).get();
	min = Stream.of(14, 35, -7, 46, 98).min(Integer::compare).get();
	min = Stream.of(14, 35, -7, 46, 98).mapToInt(Integer::new).min();
	
	int max = Stream.of(14, 35, -7, 46, 98).reduce(Integer::max).get();
	max = Stream.of(14, 35, -7, 46, 98).max(Integer::compare).get();
	max = Stream.of(14, 35, -7, 46, 98).mapToInt(Integer::new).max();
   ```
    
9. 并行处理

	```
	long result = dataList.parallelStream().mapToInt(line -> processItem(line)).sum();
	```
    
10. 集合上的各种查询(LINQ in Java)

	```
	List<Album> albums = Arrays.asList(unapologetic, tailgates, red);
	
	//筛选出至少有一个track评级4分以上的专辑，并按照名称排序后打印出来。
	albums.stream()
	  .filter(a -> a.tracks.stream().anyMatch(t -> (t.rating >= 4)))
	  .sorted(comparing(album -> album.name))
	  .forEach(album -> System.out.println(album.name));
	
	//合并所有专辑的track
	List<Track> allTracks = albums.stream()
	  .flatMap(album -> album.tracks.stream())
	  .collect(toList());
	
	//根据track的评分对所有track分组
	Map<Integer, List<Track>> tracksByRating = allTracks.stream()
	  .collect(groupingBy(Track::getRating));
	```
	
原文: <https://github.com/aruld/java-oneliners/wiki>

---

***补充 by 飒然Hang***: 上述的**单行代码编程**确实能够减少代码的字符数，也经常能够给人以高大上的感觉，但是在Java编程中字符其实是非常廉价的，尤其是现在诸如Intellij等IDE已经具有自动补充/生成代码、重构等智能化功能。如果仅仅是为了减少字符的数量，那么没必要刻意去追求**单行代码编程**。让你的代码易于阅读才是最关键的。