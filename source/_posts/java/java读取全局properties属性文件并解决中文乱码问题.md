---
title: java读取全局properties属性文件并解决中文乱码问题
date: 2015-8-19 18:16:24
categories: 
- Java
tags:
---

# java读取全局properties属性文件并解决中文乱码问题


```java
	import org.apache.log4j.Logger;

	import java.io.BufferedReader;
	import java.io.IOException;
	import java.io.InputStream;
	import java.io.InputStreamReader;
	import java.util.Properties;

	public class Config {

	    private static Logger logger = Logger.getLogger(Config.class);

	    public class Key {
	        public static final String SmsPattern = "sms.pattern";
	    }

	    public Config(){}
	    private static Properties props = new Properties();
	    static{
	        try {
	            InputStream inputStream = Thread.currentThread().getContextClassLoader().getResourceAsStream("../config.properties");
	            BufferedReader bf = new BufferedReader(new InputStreamReader(inputStream));
	            props.load(bf);
	            logger.info("config load success.");
	        } catch (IOException e) {
	            logger.fatal(e);
	        }
	    }
	    public static String getValue(String key){
	        return props.getProperty(key);
	    }
	}
```