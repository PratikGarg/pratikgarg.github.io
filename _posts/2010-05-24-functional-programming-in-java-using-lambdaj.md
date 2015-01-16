---
layout: post
header-img: img/default-blog-pic.jpg
author: PGarg
description: 
post_id: 3527
created: 2010/05/24 10:31:25
created_gmt: 2010/05/24 05:31:25
comment_status: open
image: /assets/article_images/2014-07-06-oracle/desktop.jpg

---

# Functional Programming in Java Using "lambdaj"

The new JVM based Post Modern Functional Languages (as creator of Scala, Martin Ordesky named them) like Scala and JRuby which are influenced from both Object Oriented and Functional programming languages  have gone far ahead in the terms of functionality that they offer as compared to Java.
    
    Apart from the object richness, these languages are loaded with most characteristics of a FL like (first class functions, closures, traits static/dynamic type inference, lazy evaluation, etc) which helps in bringing more modularity to the code and from aesthetics perspective these languages are less verbose and more readable as compared to java.
    
    Few people have put efforts and wrote libraries which can help in leveraging some of these features in Java too. The motivation is not to compete with scala or ruby as they are far ahead. It’s like if you don’t have the option to switch over to these languages (especially scala) or you are still waiting for the Java7 release.
    
    The ones which I have been using are <a href="http://code.google.com/p/lambdaj/">lambdaj</a> written by Mario Fusco and <a href="http://www.jroller.com/ghettoJedi/entry/enumerable_java_0_2_2">Enumerable</a> written by Hakan Raberg. Others I am aware of but not used are <a href="http://functionalj.sourceforge.net/">FunctionalJ</a> and <a href="file:///C:/Users/Pratik/Desktop/functionaljava.org/">FunctionalJava</a> .Most of them provide a similar set of features and differ bit in syntax , it only like the one to which you get accustomed too.
    

lambdaj offers more features compared to Enumerable. Just include the jar and it works, since it uses the proxies for the implementation it’s a bit slow. Enumerable takes different approach for closure support it uses ASM byte code manipulation to capture expressions as closures. So you have to perform an extra simple step of compiling the lamdas before hand.
    
    
    Lets dive into some of the code snippets that we regularly write in java and see how they can be made concise and readable using these libraries.
    
    <span style="text-decoration: underline;">Constructs for Collections</span> - Iterate through a Collection<String> and return a collection<String> starting with "inv"
    
    <em>Using Java</em>
    
    [sourcecode lang="java"]
    List&lt;String&gt; list = Arrays.asList(&quot;invalid1&quot;, &quot;invalid2&quot;);
    
    List&lt;String&gt; filteredList = new ArrayList&lt;String&gt;();
    
    for(String string : list) {
    
      if(string.startsWith(&quot;inv&quot;))
    
        filteredList.add(string);
    
    }
    return filteredList;
    
    [/sourcecode]
    
    <em>Using lambdaj</em>
    [sourcecode lang="java"]
     List&lt;String&gt; list = Arrays.asList(&quot;invalid1&quot;, &quot; invalid2&quot;);
    
    return filter(startsWith(&quot;inv&quot;), list);
    [/sourcecode]
    <em>Using Enumerable</em>
    [sourcecode lang="java"]
    List&lt;/span&gt; list = Arrays.asList(&quot;invalid1&quot;, &quot; invalid2&quot;);
    
    return Collect(list,fn(s,s.startsWith(&quot;inv&quot;));
    [/sourcecode]
    Filter/Collect are static methods provided in the lambdaj/Enumerable library which do the actual filtering/collection on collection. "<strong>startsWith"</strong> is expressed as a hamcrest matcher and the lambda expression fn(s,s.startsWith("inv")) will be converted to below , after you do the lambda expression weaving.
    [sourcecode lang="java"]
    new Fn1() {
    public Object call(Object arg) {
    return (String) arg.startsWith(&quot;inv&quot;);
    }};
    [/sourcecode]
    
    In this case the 6 lines of code are just reduced to 1.
    

Closures
    
    
    We have a scenario where you have search for an employee, compare its name with some pattern and if it matches, send the message. Now gradually you can have more criteria on which you want to match, you want a regex match or don’t want to use ignore case and many more.  For all those cases you will end up duplicating the sendMessage() method with different line instead of the one shown marked with dashes(-).
    
    <em>Using Java</em><em> </em>
    [sourcecode lang="java"]
    public boolean  sendMessage(String pattern) {
    
        FinderService service = context.getService(&quot;finderService&quot;);
    
        boolean found = false;
    
            try {
    
            Employee employee = service.find();
    
            if (null != employee) {
    
        --------  if (employee.getName().equalsIgnoreCase(pattern)) -------
               found = true;
    
            }
    
            if(found)  service.sendMessage(&quot;Found the employee&quot;);
    
           } catch (Exception e) {
    
             found = false;
    
           } 
        return found;  
    }
    
    
    [/sourcecode]
    <em>Using lambdaj</em><em> </em>
    [sourcecode lang="java"]
    public boolean sendMessageUsingIgnoreCase() {
    
        Closure2&lt;Employee, String&gt; ignoreCaseClosure  = closure(Employee.class, String.class);
    
        {of(this).ignore(var(Employee.class), var(String.class));}
    
        return sendMessage(&quot;pattern&quot;, ignoreCaseClosure);
    
    }
    
    
    
    public boolean sendMessageUsingRegexCriteria() {
    
        Closure2&lt;Employee, String&gt; regexClosure   =closure(Employee.class,String.class);
    
        {of(this).regex(var(Employee.class), var(String.class));}
    
        return sendMessage(&quot;pattern&quot;, regexClosure);
    }
    
    
    
    private boolean ignore(Employee emp, String pattern) {
    
        return emp.getName().equalsIgnoreCase(pattern);
    }
    
    
    
    private boolean regex(Employee employee, String pattern) {
    
        Pattern p = Pattern.compile(pattern);
    
        return p.matcher(employee.getName()).find();
    
    }
    
    
    
    private boolean sendMessage(String pattern,Closure2&lt;Employee, String&gt; closure) {
    
            FinderService service = context.getService(&quot;finderService&quot;);
    
            boolean found = false;
    
                try {
    
                Employee employee = service.find();
    
                if (null != employee) {
    
        ----------  if ((Boolean) closure.apply(employee, pattern)){ ---------
                   found = true;
    
                }
    
                if(found)  service.sendMessage(&quot;Found the employee&quot;);
    
               } catch (Exception e) {
    
                 found = false;
    
               } 
            return found;  
    }
    [/sourcecode]
    We created two closures ignoreCaseClosure and regexClosure which are basically the ignore and regex methods that will be used inside the sendMessage() method . This was you can create as many closures as you want based on different criteria’s and pass them to sendMessage().Here we extracted the logic which is variable and avoid duplication.
    
    
    
    <span style="text-decoration: underline;">Matchers</span>
    
    Consider this code snippet in java, which iterates though a collection and creates a concatenation string in the format "key = Value &amp; key = value &amp;...".While processing it ignores the entries which have keys starting with "graphType" or "type".
    
    <em>Using Java</em>
    [sourcecode lang="java"]
    for (Entry&lt;String, String&gt; param : params.entries()) {
    
        if (!&quot;graphPage&quot;.equals(param.getKey()) &amp;&amp; !&quot;type&quot;.equals(param.getKey())) {
    
            if (sb == null) {
    
                sb= &lt;strong&gt;new&lt;/strong&gt; StringBuffer();
    
        } else {
    
        sb.append('&amp;');
    
    }
    
    sb.append(param.getKey()).append('=').append(param.getValue());
    
    [/sourcecode]
    
    <em>Using lambdaj </em><em> </em>
    [sourcecode lang="java"]
    OrMatcher&lt;String&gt; toExclude = or(equalTo(&quot;type&quot;), equalTo(&quot;graphPage&quot;));
    
    List&lt;Entry&lt;String, String&gt;&gt; paramsList = select(params.entries(),  not(having(on(Entry.class).getKey(), toExclude)));
    
    String graphLink = join(paramsList, &quot;&amp;&quot;);
    [/sourcecode]
    
    The code is almost self explanatory you create an orMatcher for the types to be excluded. Then you apply select on the list using that matcher which return you a List<Entries>. Finally a join on it to return a string of concatenated entries.
    
    There are more features available with these libraries which are quite useful. One way of using them in you application is to create a seperate maven project with the available source code of library and then use it. This is also help in modifying/adding new stuff to the library as per project requirements.