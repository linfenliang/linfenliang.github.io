---
layout: post
title:  "Mac下Maven以jetty方式启动web项目报错Permission Denied"
subtitle: ""
date:   2017-06-28
header-img: "img/post-bg-tech.jpg"
tags:
    - 技术学习 maven jetty
categories: study
---
# 报错信息

	[ERROR] Failed to execute goal org.eclipse.jetty:jetty-maven-plugin:9.4.6.v20170531:run (default-cli) on project pscq-web: Failure: Permission denied -> [Help 1]
	[ERROR]
	[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
	[ERROR] Re-run Maven using the -X switch to enable full debug logging.
	[ERROR]
	[ERROR] For more information about the errors and possible solutions, please read the following articles:
	[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException

# 分析

大致来看，是在run时，被告知没有权限,查看相关配置发觉没有什么问题。

最后在GITHUB上一篇文章，说80端口只能是root账号才有权限占用。

# 项目关键配置


		<plugin>
			<groupId>org.eclipse.jetty</groupId>
			<artifactId>jetty-maven-plugin</artifactId>
			<version>9.4.6.v20170531</version>
			<configuration>
			<httpConnector>
				<port>80</port>
			</httpConnector>

				<webAppConfig>
					<contextPath>/pscq</contextPath>
				</webAppConfig>
				<scanIntervalSeconds>5</scanIntervalSeconds>
				<systemProperties>
					<systemProperty>
						<name>org.mortbay.util.URI.charset</name>
						<value>${project.build.sourceEncoding}</value>
					</systemProperty>
				</systemProperties>
			</configuration>
		</plugin>

# 解决办法

由于Mac下默认的用户是 linfenliang，解决办法要么是更改为root，要么是更换端口，我更换成了8080端口，项目正常启动

# 参考

[Execute maven plugin goal jetty:run on Intellij error: “Permission denied”
](https://stackoverflow.com/questions/8531862/execute-maven-plugin-goal-jettyrun-on-intellij-error-permission-denied)
