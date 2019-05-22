---
title: Swagger详解
date: 2018-10-27 14:16:49
tags: Swagger
categories: javaweb
---

Swagger注解使用说明

- @Api

  这个注解用来生命此类为Swagger resource API ，只有被这个@Api注解的类才能被Swagger扫描到；但是我发现这个不是必要的，不要这个注解有其它条件也可以扫描到类下面的Api

- @ApiOperation

  这个注解用来声明单个方法为一个Api接口，其中**value**是用来给这个Api作简短的介绍；**notes**	允许你给出重要的和更详细的该接口的信息；**response**和**responseContainer**主要是用来展示返回示例的，如果在此使用了这两个就会体现在Example Value上

- @ApiResponses、@ApiResponse

  这两个注解组合使用在类上，是为了展示方法返回状态码的含义

- @ApiParam

  此注解就是用来显示请求该接口需要哪些参数,这个用来GET方法上

- @ApiImplicitParam、@ApiImplicitParam

  此注解也是用来显示接口参数，但是是用在参数放在Body里面的参数

- @ApiModel、@ApiModelProperty

  这两个注解是用来在请求参数中如果有对象，可以用这两个注解来具体描述里面的参数