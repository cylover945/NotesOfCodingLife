# Java8中Lambda表达式的使用

	下面是Java8中lambda表达式使用的demo，代码简洁，有助于理解Java函数式编程的用法。

## 1. lambda实现Runnable

		//Java8之前
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("lambda实现Runnable");
            }
        }).start();
        //Java8
        new Thread(()-> System.out.println("lambda实现Runnable")).start();

## 2. lambda循环遍历

		String[] nba = {"James", "Wade", "Carmelo", "Paul"};
        List<String> players = Arrays.asList(nba);
        // 以前的foreach循环
        for (String player : players) {
            System.out.println("foreach循环"+player);
        }

        // 使用 lambda 表达式以及函数操作
        players.forEach((player) -> System.out.println("lambda表达式"+player));

        // 在 Java 8 中使用双冒号操作符
        players.forEach(System.out::println);

## 3. lambda集合排序

		String[] nba = {"James", "Wade", "Carmelo", "Paul"};

        //使用匿名内部类 根据 名字长度 排序
        Arrays.sort(nba, new Comparator<String>() {
            @Override
            public int compare(String s1, String s2) {
                return s1.length()-s2.length();
            }
        });
        //使用lambda表达式 根据 名字长度 排序
        Arrays.sort(nba,(String s1,String s2)->(s1.length()-s2.length()));

	同理可以使用名字首字母排序

		//使用匿名内部类 根据 首字母 排序
        Arrays.sort(nba, new Comparator<String>() {
            @Override
            public int compare(String s1, String s2) {
                return s1.compareTo(s2);
            }
        });
        //使用lambda表达式 根据 首字母 排序
        Arrays.sort(nba,(String s1,String s2)->(s1.compareTo(s2)));

	排序的种类还有很多，就不一一举例，可以根据实际情况编写合理的排序方式

## 4. lambda使用Predicate过滤
代码示例

		String[] nba = {"James", "Wade", "Carmelo", "Paul"};
        List<String> names = Arrays.asList(nba);
        //添加过滤条件，名字以J开头
        Predicate<String> startsWithJ = (n) -> n.startsWith("J");
        //添加过滤条件，名字4个字母长
        Predicate<String> fourLetterLong = (n) -> n.length() == 4;
        names.stream()
                .filter(startsWithJ.or(fourLetterLong))
                .forEach((n) -> System.out.print("名字以J开头或者是4个字母长的名字是 : " + n+"\n"));

输出结果

		名字以J开头或者是4个字母长的名字是 : James
		名字以J开头或者是4个字母长的名字是 : Wade
		名字以J开头或者是4个字母长的名字是 : Paul

## lambda和Stream结合使用

+ **Stream处理集合，并生成新的集合**

	需要注意的是各个方法的使用顺序，如：filter和sorted先后顺序会产生不同结果
	
		String[] nba = {"James", "Wade", "Carmelo", "Paul","Bron"};
        List<String> names = Arrays.asList(nba);
        //添加过滤条件，名字以J开头
        Predicate<String> startsWithJ = (n) -> n.startsWith("J");
        //添加过滤条件，名字4个字母长
        Predicate<String> fourLetterLong = (n) -> n.length() == 4;
        List<String> list = names.stream()
                .filter(startsWithJ.or(fourLetterLong))//过滤
                .limit(4)//个数限制
                .sorted((String s1,String s2)->(s1.compareTo(s2)))//字母排序
                .collect(Collectors.toList());
        list.forEach(System.out ::println);

+ **Stream进行MapReduce**

	get获取到经过处理后想要得到的数据

		// 为每个商品涨价12%，计算出总卖价
        // 老方法：
        List<Integer> costBeforeTax = Arrays.asList(100,200,300,400,500);
        double total = 0;
        for (Integer cost : costBeforeTax) {
            double price = cost + .12 * cost;
            total = total + price;
        }
        System.out.println("Total : " + total);

        // 使用map reduce：
        double bill = costBeforeTax.stream()
                .map((cost) -> cost + .12 * cost)
                .reduce((sum, cost) -> sum + cost)
                .get();
        System.out.println("Total : " + bill);

	sum直接求和

		List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        int total = numbers.stream()
                .mapToInt((x)->x)
                .sum();
        System.out.println(total);

	summaryStatistics对元素的数据汇总

		//计算 count, min, max, sum, and average for numbers
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        IntSummaryStatistics stats = numbers
                .stream()
                .mapToInt((x) -> x)
                .summaryStatistics();

        System.out.println("List中最大的数字 : " + stats.getMax());
        System.out.println("List中最小的数字 : " + stats.getMin());
        System.out.println("所有数字的总和   : " + stats.getSum());
        System.out.println("所有数字的平均值 : " + stats.getAverage());


## 总结

		lambda表达式使代码看上去更简洁，与此同时，应该合理的使用，让团队开发更有效率。