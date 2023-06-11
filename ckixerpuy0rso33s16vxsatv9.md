---
title: "How to migrate static DB configurations between environments"
datePublished: Sun Dec 20 2020 17:30:47 GMT+0000 (Coordinated Universal Time)
cuid: ckixerpuy0rso33s16vxsatv9
slug: how-to-migrate-static-db-configurations-between-environments
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1608485422226/XQxz8H-Bo.jpeg
tags: spring, java, databases

---

I would like to be able to setup my data on the QA environment (typically from some kind of back-office app), perform some testing and move it easily to STAGE / PRE-PROD without manually performing the configuration again and again. You must have heard this requirement many times before, so do I. Of course we are not talking about the whole database or any type of stateful data. In most of the cases it's some sort of static configuration stored in limited set of database tables which doesn't change often once it's setuped and it's pretty much the same among different environments. An example might be some Product Catalogue. We have the same set of products among ours envs, so once we configure couple of new products, we test our business logic and we move the data ahead to the next staging environment. Here are some of the possible approaches when dealing with such requirement.

## Using database migration tool

This is the easiest possible approach, especially when you already have such tool in place. Common examples for DB migration tools are Liquibase and Flyway. Their primary goal is to provide an SQL agnostic approach via configuring SQL scripts via XML or YAML syntax, which is then easily applied with their corresponding Maven or Gradle plugins. Note that DB migration tools is good-to-have in general, it is the easiest way for migrating data. The point here is more on the specific use-case, as described. So here is an example for Liquibase YAML INSERT script, taken from the official Liquibase documentation:


```
changeSet:  
  id:  insert-statement  
  author:  liquibase-docs  
  changes:  
  -  insert:  
      catalogName:  cat  
      columns:  
      -  column:  
          name:  address  
          value:  address value  
      dbms:  '!h2,  mysql'  
      schemaName:  public  
      tableName:  person
``` 

Once you prepare your statements, they are being translated to the underlying SQL server with the corresponding syntax:


```
INSERT  INTO  cat.person  (address)  VALUES  ('address value');
``` 

You can easily configure your DB migration tool to support profiles within Maven/Gradle, giving you the freedom to execute your statements against different environments easily.

**Pros: **

 - full control over your configuration and statements, meaning that only devs can execute the scripts and see what is being executed

**Cons:  **

 - requires development effort, which makes it time consuming along with plain annoying for the developers. Business don't want to pay/allocate efforts for boilerplate work. Developers don't want to bother with such repetitive, operational work

 - huge possibility of mistakes (typos). Especially when developers write one and the same script multiple times. They just want to do and forget

## Export/import functionality

This is a solution which does the job very well, especially when the process is

*I configure my static data, I test it, I go to the next environment and move my data with minimum to zero efforts, repeat.
*

Adding an Export button in our backoffice, along with Import, makes it possible to move our configuration with two clicks. The first example for such solution is exporting and importing our data to/from binary files, using Spring MVC endpoints:


```
// Export

	@PostMapping(value = "/export")
	public void export(HttpServletResponse response,
			@RequestParam("productIds") ArrayList<Long> selectedProductsIds) throws IOException {
		List<Product> products = (List<Product>) productRepository.findAll(selectedProductsIds);
		response.setContentType("application/octet-stream; charset=utf-8");
		response.setHeader("Content-disposition", "attachment; filename=products-" + new Date().getTime() + ".dat");
		OutputStream outputStream = response.getOutputStream();
		outputStream.write(SerializationUtils.serialize(products));
		outputStream.flush();
		outputStream.close();
	}
``` 

```
// Import

	@PostMapping(value = "/import")
	public String import(@RequestParam("file") MultipartFile file, RedirectAttributes redirectAttributes, Model model)
					byte[] bytes = file.getBytes();
		ByteArrayInputStream fileIn = new ByteArrayInputStream(bytes);
        ObjectInputStream in = new ConfigurableObjectInputStream(fileIn,
                Thread.currentThread().getContextClassLoader());
        List<Product> products= (List<Product>) in.readObject();
		initModel(model);
		model.addAttribute("products", products);
		return "products";
	}
``` 

This implementation has very specific purpose - whoever performs this data migration might change the data in between (by mistake), because after all it is a file which is manually transmitted. With exporting and importing in binary, we make sure that this file is hard to change by hand, especially by non-technical people. We can do the same thing in CSV, XML or other serialization format as well, but if you want to guard it from change in between, you might want to use similar approach.

**Pros:**

 - easier to work with, especially for non-technical staff members

 - move data with minimal efforts

 - eliminating development time allocation

**Cons**

 - there is still limited chance for mistakes

## E2E migration solution

We can go even further and implement more user-friendly and easy-to-use solution with providing E2E tool as part of our backoffice, which enables operations/customers/administrators to configure and apply the data on different environments using either DB migration tool integration or directly using the persistence ORM layer with switching the DB profile from the frontend.

## Conclusion

This article briefs couple of possible approaches when dealing with common situation of being able to migrate certain static config between our low environments with minimal efforts. Of course, there are a lot of variables - the architecture of our system, the DB model, the tech stack, the requirements. Still, have in mind those ideas if you face similar use-case in your daily work.

WARNING: PROD data is always migrated manually in controlled environment, I am not considering any PROD data migration approaches here.









