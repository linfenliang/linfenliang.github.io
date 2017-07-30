---
layout: post
title:  "spring-data整合MongoDB代码测试"
subtitle: ""
date:   2017-07-03
header-img: "img/post-bg-tech.jpg"
tags:
    - 技术学习 mongodb
categories: study
---

# 场景

目前业务场景中有用MySQL存储车辆轨迹数据以及温度计温湿度数据，数据增长非常快,每个月有大概200万到300万的增长速度，并且随着下一步接入企业的进一步增加，温度计数据与车辆轨迹数据还会持续增加（业务需求，每辆车都必须要安装GPS），之前做过数据分库分表，数据库MySQL分库分表导致运维查询会很麻烦，并且不利于数据的统计分析。
目前的业务需求是希望数据稳定的持续的存储以及能够实现实时查询功能，并能够为将来数据的统计分析做好准备，同时数据包含GIS信息，后期可能会需要按照GIS查询。

# 实现思路
## 数据冷热分离
目前用户查询需要的数据一般都是最近三个月以内的，历史数据查询非常少，可以据此将数据分离，最近三个月的数据放到MySQL中处理(目前数据每月增量为500万、800万)，设计好分表存储即可，三个月以上的数据则存储到MongoDB中，考虑到数据查询三个月临界点查询问题，这里将两个月以上的数据存储到MongoDB中（第三个月的数据MySQL与MongoDB中均有一份）

前期将车辆轨迹（含温湿度）数据、仓库温湿度数据、温度计温湿度数据按月批量从MySQL转储到MongoDB中，用户查询时按照时间进行判断路由到MySQL或MongoDB中，从而实现基于MongoDB的数据统计分析与实时查询。

## 数据实时查询

由于预估随着接入企业的增多，数据规模会持续增大，MySQL三个月数据的存储依然压力较大，故后期需要实现MongoDB的实时查询与实时存储。



# 准备工作

MongoDB的基本用法、集群、高可用（自动故障转移）等，在我的博客之前有介绍，详细参考GitHub [MongoDB](https://github.com/linfenliang/mongodb)

## MongoDB的安装

这里测试用的是Mac下基于Docker的MongoDB single instance

安装步骤与[安装RabbitMQ](http://localhost:3000/work/messagequeue/2017/04/22/rabbitmq-docker/)类似，可参考 DockerStore下的https://store.docker.com/images/mongo

运行MongoDB：
docker run --name pscq-mongo -p 27017:27017 -p 28017:28017 -d mongo

如果已经下载并运行过，则运行命令：
docker run -p 27017:27017 -p 28017:28017 -d mongo
即可。

# 代码实战

## POM添加MongoDB依赖


    <dependency>
        <groupId>org.springframework.data</groupId>
        <artifactId>spring-data-mongodb</artifactId>
    </dependency>

## 用于存储的实体类，注意添加了索引

    package com.springboot.data.mongodb.domain;

    import java.math.BigInteger;
    import java.util.Date;

    import org.springframework.data.annotation.Id;
    import org.springframework.data.mongodb.core.index.CompoundIndex;
    import org.springframework.data.mongodb.core.index.CompoundIndexes;
    import org.springframework.data.mongodb.core.index.Indexed;
    import org.springframework.data.mongodb.core.mapping.Document;
    import org.springframework.data.mongodb.core.mapping.Field;

    import com.springboot.data.mongodb.common.domain.BaseEntity;

    /**
     *
     * @Author linfl
     * @Version 1.0
     * @Date 2017年3月9日
     */
    @Document
    @CompoundIndexes({@CompoundIndex(name="vehicleIdGatherTime_idx",def="{'vehicleId':1,gatherTime:1}")})
    public class VehicleOnlineData extends BaseEntity{

    	@Id
    	@Indexed
    	private BigInteger id;
    	@Indexed
    	private String vehicleId;
    	@Indexed
    	@Field("plateNo")//指定数据库存储字段
    	private String vehicleNo;
    	private Date gatherTime;
    	private Integer gpsSpeed;
    	private Double realLongitude;
    	private Double realLatitude;
    	private Double offsetLongitude;
    	private Double offsetLatitude;
    	private Integer direct;
    	private Integer altitude;
    	private Integer accStatus;
    	private Double gForce;
    	private Integer thermometerProbe1;
    	private Integer thermometerProbe2;
    	private Integer thermometerProbe3;
    	private Integer thermometerProbe4;
    	private Double humidometerProbe1;
    	private Double humidometerProbe2;
    	private Double humidometerProbe3;
    	private Double humidometerProbe4;
    	@Indexed
    	private Date updateDate;
    	@Indexed
    	private Date createDate;
    	@Indexed
    	private String delFlag;
    	private String createBy;
    	private String updateBy;
    	private String remarks;
    	public BigInteger getId() {
    		return id;
    	}
    	public void setId(BigInteger id) {
    		this.id = id;
    	}
    	public String getVehicleId() {
    		return vehicleId;
    	}
    	public void setVehicleId(String vehicleId) {
    		this.vehicleId = vehicleId;
    	}
    	public String getVehicleNo() {
    		return vehicleNo;
    	}
    	public void setVehicleNo(String vehicleNo) {
    		this.vehicleNo = vehicleNo;
    	}
    	public Date getGatherTime() {
    		return gatherTime;
    	}
    	public void setGatherTime(Date gatherTime) {
    		this.gatherTime = gatherTime;
    	}
    	public Integer getGpsSpeed() {
    		return gpsSpeed;
    	}
    	public void setGpsSpeed(Integer gpsSpeed) {
    		this.gpsSpeed = gpsSpeed;
    	}
    	public Double getRealLongitude() {
    		return realLongitude;
    	}
    	public void setRealLongitude(Double realLongitude) {
    		this.realLongitude = realLongitude;
    	}
    	public Double getRealLatitude() {
    		return realLatitude;
    	}
    	public void setRealLatitude(Double realLatitude) {
    		this.realLatitude = realLatitude;
    	}
    	public Double getOffsetLongitude() {
    		return offsetLongitude;
    	}
    	public void setOffsetLongitude(Double offsetLongitude) {
    		this.offsetLongitude = offsetLongitude;
    	}
    	public Double getOffsetLatitude() {
    		return offsetLatitude;
    	}
    	public void setOffsetLatitude(Double offsetLatitude) {
    		this.offsetLatitude = offsetLatitude;
    	}
    	public Integer getDirect() {
    		return direct;
    	}
    	public void setDirect(Integer direct) {
    		this.direct = direct;
    	}
    	public Integer getAltitude() {
    		return altitude;
    	}
    	public void setAltitude(Integer altitude) {
    		this.altitude = altitude;
    	}
    	public Integer getAccStatus() {
    		return accStatus;
    	}
    	public void setAccStatus(Integer accStatus) {
    		this.accStatus = accStatus;
    	}
    	public Double getgForce() {
    		return gForce;
    	}
    	public void setgForce(Double gForce) {
    		this.gForce = gForce;
    	}
    	public Integer getThermometerProbe1() {
    		return thermometerProbe1;
    	}
    	public void setThermometerProbe1(Integer thermometerProbe1) {
    		this.thermometerProbe1 = thermometerProbe1;
    	}
    	public Integer getThermometerProbe2() {
    		return thermometerProbe2;
    	}
    	public void setThermometerProbe2(Integer thermometerProbe2) {
    		this.thermometerProbe2 = thermometerProbe2;
    	}
    	public Integer getThermometerProbe3() {
    		return thermometerProbe3;
    	}
    	public void setThermometerProbe3(Integer thermometerProbe3) {
    		this.thermometerProbe3 = thermometerProbe3;
    	}
    	public Integer getThermometerProbe4() {
    		return thermometerProbe4;
    	}
    	public void setThermometerProbe4(Integer thermometerProbe4) {
    		this.thermometerProbe4 = thermometerProbe4;
    	}
    	public Double getHumidometerProbe1() {
    		return humidometerProbe1;
    	}
    	public void setHumidometerProbe1(Double humidometerProbe1) {
    		this.humidometerProbe1 = humidometerProbe1;
    	}
    	public Double getHumidometerProbe2() {
    		return humidometerProbe2;
    	}
    	public void setHumidometerProbe2(Double humidometerProbe2) {
    		this.humidometerProbe2 = humidometerProbe2;
    	}
    	public Double getHumidometerProbe3() {
    		return humidometerProbe3;
    	}
    	public void setHumidometerProbe3(Double humidometerProbe3) {
    		this.humidometerProbe3 = humidometerProbe3;
    	}
    	public Double getHumidometerProbe4() {
    		return humidometerProbe4;
    	}
    	public void setHumidometerProbe4(Double humidometerProbe4) {
    		this.humidometerProbe4 = humidometerProbe4;
    	}
    	public Date getUpdateDate() {
    		return updateDate;
    	}
    	public void setUpdateDate(Date updateDate) {
    		this.updateDate = updateDate;
    	}
    	public Date getCreateDate() {
    		return createDate;
    	}
    	public void setCreateDate(Date createDate) {
    		this.createDate = createDate;
    	}
    	public String getDelFlag() {
    		return delFlag;
    	}
    	public void setDelFlag(String delFlag) {
    		this.delFlag = delFlag;
    	}
    	public String getCreateBy() {
    		return createBy;
    	}
    	public void setCreateBy(String createBy) {
    		this.createBy = createBy;
    	}
    	public String getUpdateBy() {
    		return updateBy;
    	}
    	public void setUpdateBy(String updateBy) {
    		this.updateBy = updateBy;
    	}
    	public String getRemarks() {
    		return remarks;
    	}
    	public void setRemarks(String remarks) {
    		this.remarks = remarks;
    	}



    }

## 持久化接口（基于MongoRepository）

    package com.springboot.data.mongodb.dao;

    import java.util.Date;

    import org.springframework.data.domain.Page;
    import org.springframework.data.domain.Pageable;
    import org.springframework.data.mongodb.repository.MongoRepository;
    import org.springframework.data.mongodb.repository.Query;

    import com.springboot.data.mongodb.domain.VehicleOnlineData;

    /**
     *
     * @Author linfl
     * @Version 1.0
     * @Date 2017年3月9日
     */
    public interface VehicleOnlineDataRepository extends MongoRepository<VehicleOnlineData, String> {
    //CrudRepository<VehicleOnlineData, String>
    //	Long countByVehicleId(String vehicleId);
    //	List<VehicleOnlineData> findByVehicleId(String vehicleId);
    //	Long countByVehicleId(String vehicleId);
    	@Query("{ 'vehicleId':?0, 'gatherTime': {$gte:?1,$lt:?2}}")
    	Page<VehicleOnlineData> findByVehicleIdAndGatherTimeRange(String vehicleId,Date gatherTimeBegin,Date gatherTimeEnd,Pageable pageable);
    //	@Query("{ 'gatherTime': {$gte:?1,$lt:?2}}")
    //	Page<VehicleOnlineData> findByMatchVehicleId(String vehicleId,Date gatherTimeBegin,Date gatherTimeEnd,Pageable pageable);

    //	void saveData(VehicleOnlineData entity);
    }

## 基于SpringBoot的启动类

    package com.springboot.data.mongodb;

    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;


    /**
     * Hello world!
     *
     */
    @EnableAutoConfiguration
    @SpringBootApplication
    public class MongodbApp extends WebMvcConfigurerAdapter{
    	public static void main(String[] args) throws Exception {
    		SpringApplication.run(MongodbApp.class, args);
    	}

    }

## 单元测试

    package com.springboot.data.mongodb;

    import static org.junit.Assert.assertTrue;

    import java.util.ArrayList;
    import java.util.Calendar;
    import java.util.Date;
    import java.util.List;
    import java.util.concurrent.ExecutionException;
    import java.util.concurrent.ExecutorService;
    import java.util.concurrent.Executors;
    import java.util.concurrent.Future;
    import java.util.concurrent.atomic.AtomicInteger;

    import org.apache.commons.lang3.RandomStringUtils;
    import org.apache.commons.lang3.RandomUtils;
    import org.apache.commons.lang3.builder.ReflectionToStringBuilder;
    import org.junit.Test;
    import org.junit.runner.RunWith;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.context.SpringBootTest;
    import org.springframework.data.domain.Page;
    import org.springframework.data.domain.PageRequest;
    import org.springframework.data.domain.Sort;
    import org.springframework.data.domain.Sort.Direction;
    import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
    import org.springframework.util.StopWatch;

    import com.springboot.data.mongodb.MongodbApp;
    import com.springboot.data.mongodb.dao.VehicleOnlineDataRepository;
    import com.springboot.data.mongodb.domain.VehicleOnlineData;

    /**
     * Unit test for simple App.
     */
    @RunWith(SpringJUnit4ClassRunner.class)
    @SpringBootTest(classes=MongodbApp.class)
    //@Transactional
    public class MongodbAppTest {
    	@Autowired
    	private VehicleOnlineDataRepository repository;
    //	@Autowired
    //	private MongoTemplate mongoTemplate;
    	static Date gatherTimeBegin;
    	static Date gatherTimeEnd;
    	String vehicleId = "98765432109876543210987654321747";
    	static{
    		Calendar cal = Calendar.getInstance();
    		cal.set(2017, 06, 04, 11, 20, 00);
    		cal.set(Calendar.MILLISECOND,000);
    		gatherTimeBegin = cal.getTime();
    		cal.set(2017, 06, 04, 11, 20, 50);
    		cal.set(Calendar.MILLISECOND,000);
    		gatherTimeEnd = cal.getTime();
    	}


    	/**
    	 * Rigourous Test :-)
    	 */
    	@Test
    	public void testApp() {
    		assertTrue(true);
    	}
    //	@Test
    //	public void mongoTemplateTest(){
    //		StopWatch watch = new StopWatch();
    //		watch.start("mongotemplate query");
    //		Query condition = new Query(Criteria.where("vehicleId").is(vehicleId)
    //				//.andOperator
    //				).addCriteria(
    //				(Criteria.where("gatherTime").gte(gatherTimeBegin).lt(gatherTimeEnd))
    //				).with(new PageRequest(0, 10,new Sort(Direction.DESC,"gatherTime"))).limit(10);
    //		List<VehicleOnlineData> page = mongoTemplate.find(condition, VehicleOnlineData.class);
    //		System.out.println(page);
    //		page.forEach(v->{System.out.println(ReflectionToStringBuilder.toString(v));});
    //		watch.stop();
    //		System.out.println(watch.prettyPrint());
    //		watch.start("mongotemplate count");
    //		System.out.println("mongoTemplate count:"+mongoTemplate.count(condition, VehicleOnlineData.class));
    //		watch.stop();
    //		System.out.println(watch.prettyPrint());
    //	}
    	@Test
    	public void repositoryTest(){
    		StopWatch watch = new StopWatch();
    		watch.start("query by condition");
    		Page<VehicleOnlineData> page = repository.findByVehicleIdAndGatherTimeRange(vehicleId, gatherTimeBegin, gatherTimeEnd, new PageRequest(0, 10,new Sort(Direction.DESC,"gatherTime")));
    		page.forEach(v->{
    			System.out.println(ReflectionToStringBuilder.toString(v));
    		});
    		watch.stop();
    		watch.start("get page info");
    		System.out.println("repository:"+ReflectionToStringBuilder.toString(page));
    		watch.stop();
    		System.out.println(watch.prettyPrint());

    	}
    	@org.junit.Ignore
    	@Test
    	public void add1000WDataTest() throws InterruptedException, ExecutionException{
    		int inserTotal = 10_000_000;
    		final int batchSize = 5000;
    		ExecutorService es = Executors.newFixedThreadPool(8);
    		StopWatch watch = new StopWatch();
    		watch.start();
    		AtomicInteger count = new AtomicInteger((int) repository.count());
    		final int cycleSize = inserTotal/batchSize;
    		List<Future<Boolean>> futureList = new ArrayList<>();  
    		for(int i =0 ;i<cycleSize;i++){
    			futureList.add(es.submit(new Thread(){
    				@Override
    				public void run() {
    					List<VehicleOnlineData> list = new ArrayList<>();
    					for(int j=0;j<batchSize;j++){
    						list.add(getBean());
    					}
    					List<VehicleOnlineData> v = repository.save(list);
    					System.out.println(Thread.currentThread().getId()+"线程 ，总数据量：    "+count.addAndGet(batchSize)+"  ，当前保存记录数：    "+v.size());
    				}
    			},true));
    		}
    		for(Future<Boolean> f:futureList){
    			f.get();
    		}
    		watch.stop();
    		System.out.println(watch.prettyPrint());
    		es.shutdown();
    	}
    	private VehicleOnlineData getBean(){
    		VehicleOnlineData vod = new VehicleOnlineData();
    		vod.setAccStatus(1);
    		vod.setAltitude(RandomUtils.nextInt(0, 1000));
    		vod.setDirect(RandomUtils.nextInt(0, 360));
    		vod.setGatherTime(new Date());
    		vod.setGpsSpeed(RandomUtils.nextInt(0, 120));
    		vod.setHumidometerProbe1(RandomUtils.nextDouble(0.1, 0.99));
    		vod.setHumidometerProbe2(RandomUtils.nextDouble(0.1, 0.99));
    		vod.setHumidometerProbe3(RandomUtils.nextDouble(0.1, 0.99));
    		vod.setHumidometerProbe4(RandomUtils.nextDouble(0.1, 0.99));
    		vod.setOffsetLatitude(RandomUtils.nextDouble(60.1, 65.9));
    		vod.setOffsetLongitude(RandomUtils.nextDouble(110.1, 125.1));
    		vod.setRealLatitude(vod.getOffsetLatitude());
    		vod.setRealLongitude(vod.getOffsetLongitude());
    		vod.setThermometerProbe1(RandomUtils.nextInt(0, 38));
    		vod.setThermometerProbe2(RandomUtils.nextInt(0, 38));
    		vod.setThermometerProbe3(RandomUtils.nextInt(0, 38));
    		vod.setThermometerProbe4(RandomUtils.nextInt(0, 38));
    		String randomInt = RandomStringUtils.randomNumeric(3);
    		vod.setVehicleId("98765432109876543210987654321"+randomInt);
    		vod.setVehicleNo("京HM831"+randomInt);
    		vod.setCreateDate(new Date());
    		vod.setUpdateDate(vod.getCreateDate());
    		vod.setDelFlag("0");
    		vod.setCreateBy("linfenliang");
    		vod.setUpdateBy(vod.getCreateBy());
    		return vod;

    	}


    }

## 单元测试结果（效果）

    2017-07-30 16:11:02,309 [main] INFO  [ org.springframework.boot.StartupInfoLogger.logStarted(StartupInfoLogger.java:57)] - Started MongodbAppTest in 2.183 seconds (JVM running for 3.0)
    repository:org.springframework.data.domain.PageImpl@4faa298[total=0,pageable=Page request [number: 0, size 10, sort: gatherTime: DESC],content=[],pageable=Page request [number: 0, size 10, sort: gatherTime: DESC]]
    StopWatch '': running time (millis) = 92
    -----------------------------------------
    ms     %     Task name
    -----------------------------------------
    00084  091%  query by condition
    00008  009%  get page info


# MongoDB客户端管理工具

[RoboMongo](https://robomongo.org/)

![MongoDB客户端管理工具RoboMongo使用](/img/post-images/2017-06-16/201706161814.png)

# 参考

[Docker 安装 MongoDB](http://www.runoob.com/docker/docker-install-mongodb.html)

[Docker store MongoDB](https://store.docker.com/images/mongo)

[SpringData集成MongoDB](https://docs.spring.io/spring-data/mongodb/docs/1.10.6.RELEASE/reference/html/)

# 结束语

其他的测试信息我没有再贴出来，主要是实现了基于时间、ID的组合查询，数据量在1亿左右时，其组合条件查询（有索引）耗时基本在200毫秒以内。

基于GIS地理空间的查询可直接采用Spring-Data 的MongoDB的接口(MongoTemplate)，本文档未再给出具体示例代码(可参考注释掉的部分代码)。

对于MongoDB的从MySQL数据转移到MongoDB拟采用Spring-batch批操作+quartz实现，目前受限于公司服务器资源，暂未实现其生产环境使用，
