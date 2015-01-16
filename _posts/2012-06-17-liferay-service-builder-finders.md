---
layout: post
header-img: img/default-blog-pic.jpg
author: PGarg
description: 
post_id: 13715
created: 2012/06/17 14:35:52
created_gmt: 2012/06/17 09:35:52
comment_status: open
image: /assets/article_images/2014-07-06-oracle/desktop.jpg
---

# Liferay Service Builder Finders

In the last post we saw how Liferay service builder helps in generation of Services and Dao/Persistence classes which perform the basic CRUD operations. Along with the basic crud operation liferay also comes with some finder module methods using which can be used to find entities based upon one or more of the fields present in it. Let’s continue with the example of email entity to better insights on finders.

**Email Example Extended **

Suppose we want to search for emails based upon the following two criteria's 

  1. Subject matching to a particular text string
  2. Email from a particular user
Add the following xml inside the email entry declared in service.xml

[sourcecode language="xml"] <!-- Finders --> <finder return-type="Collection" name="Subject"> <finder-column name="subject"></finder-column> </finder> <finder return-type="Collection" name="From"> <finder-column name="from"></finder-column> </finder> [/sourcecode]

For the above declaration, we will see the following method in EmailPersistence and their implementation in EmailPersistenceImpl

[sourcecode language="xml"] public java.util.List<com.xebia.model.email> findBySubject( java.lang.String subject) throws com.liferay.portal.kernel.exception.SystemException;

public java.util.List<com.xebia.model.email> findByFrom( java.lang.String from) throws com.liferay.portal.kernel.exception.SystemException; [/sourcecode]

There are some additional helper methods generated for accessing data corresponding to these fields. These methods can be used to find count or limit the number of results etc.

Now an additional requirement is, to find all emails that are received on a particular date i.e. within those 24 hours. The first thing that will come to mind is the "Where" clause. But there is no out of the box facility available in liferay that will automatically generate this method. The only only solution for these type of cases is writing a custom sql query and using the finder functionality to execute this query and return the results.

** **

**Configuration**

Lets first start with writing of the custom query which will help us in retrieving all the emails between two dates. Our query will look like this

[sourcecode language="xml"] SELECT Email.* FROM Email WHERE Email.date between ? AND ? [/sourcecode]

Include this sql in /docroot/WEB-INF/src/custom-sql/default.xml file. So that default.xml sql will look something like this

Note : Portlet structure should adhere to liferay standards and we have to create a custom-sql folder inside src and then create a file default.xml

We can also create custom file and then mention the reference of that file in default.xml.

[sourcecode language="xml"] <!--?xml version="1.0" encoding="UTF-8"?--> <custom-sql> <sql id="com.xebia.service.persistence.EmailFinder.getEmailBetweenDates"> <![CDATA[ SELECT Email.* FROM Email WHERE Email.date between ? AND ? ]]> </sql> </custom-sql> [/sourcecode]

the id attribute is the unique identifier through which this query is recognized.

**Code Generation**

We will be calling this query from finder class of liferay. Following are the steps for code generation for Liferay finder classes

Create a class EmailFinderImpl in the persistence package and Extend this class from EmailPersistenceImpl

Run ant build-service. This will generate the EmailFinder interface and EmailFinderUtil

Now go to EmailFinderImpl and make it implement EmailFinder

After this add a method in EmailFinderImpl and will run service builder, the same method will be reflected in EmailFinder interface and EmailFinderUtil class as well.

The class hierarchy and structure of Finder is almost similar to the structure of services which were shown in the last blog. Following is the class diagram for Finder.

![][1]

[sourcecode language="xml"] public static String GET_EMAIL_BETWEEN_DATES = EmailFinder.class .getName() + ".getEmailBetweenDates";
    
    
    public List&lt;email&gt; getEmailBetweenDates(Date startDate,Date endDate) throws ParseException, SystemException {
    
        Session session = null;
        List&lt;email&gt; emails = null;
        try {
            session = openSession();
    
            String sql = CustomSQLUtil.get(GET_EMAIL_BETWEEN_DATES);
            SQLQuery q = session.createSQLQuery(sql);
    
            q.addEntity(&quot;Email&quot;, EmailImpl.class);
            QueryPos qPos = QueryPos.getInstance(q);
    
            qPos.add(startDate);
            qPos.add(endDate);
    
            emails = (List&lt;email&gt;) q.list();
            if (emails == null) {
                emails = Collections.emptyList();
                                     }
               } finally {
            closeSession(session);
         }
    
        return emails;
    }
    

[/sourcecode]

**Code Utilization**

Calling of the finder methods is almost same as calling any of the service methods.

[sourcecode language="xml"] List<email> emailsByDate = EmailFinderUtil.getEmailBetweenDates(Date startDate, Date endDate); [/sourcecode]

**Object Mappings**

So far we have seen how to perform cruds and advanced search on a entity. Now let’s consider this example.

Fetch all the emails that have an attachments.

At Database Level : we will have an Email table and an Attachment table with email Id as the foreign key to support one-to-many relationships.

At Object level : we will have an Email object with Collection of type attachment as one of properties.

When we retrieve the email we also expect the attachement to get populated and vice-versa during the save (Given that we have done the required relationship mappings between the entities in service.xml).

But this is not how it works in the liferay. We cannot create a relation between the entities using the service.xml which basically means no work will be done automatically either at the DB level (foreign keys) and at object level (has a relationship) , yes there are workarounds but out of the box this is not provided. In the liferay [DTD][2] for service.xml , there is section that will describe how to create one-2-one / many-2-many relationships but even if we follow that convention , service builder will not create the mappings in generated java classes. This is how the attachment entity will look like.

[sourcecode language="xml"] <entity name="Attachment" local-service="true" remote-service="false" table="Email"> <column name="id" type="long" primary="true"> <column name="type" type="String"> <column name="emailId" type="long"> <column name="content" type="String"> <!-- Finders --> <finder name="id"> <finder-column name="to"></finder-column> </finder> <finder name="emailId" return-type="Collection"> <finder-column name="from"></finder-column> </finder> </column></column></column></column></entity> [/sourcecode]

To achieve the above scenario : 1) Create an EmailDTO to contain email and its corresponding Attachment. Code below

[sourcecode language="xml"]

public EmailDTO{ private Email; private Attachment;

   [1]: http://xebee.xebia.in/wp-content/uploads/2012/05/finder_class_diag.png (finder_class_diag)
   [2]: http://www.liferay.com/dtd/liferay-service-builder_5_2_0.dtd%20