---
layout: post
title: "<번역시도>Spring 3.1 MVC example"
date: 2013-01-11 16:39
comments: true
categories: [spring framework, spring mvc, 번역]
---

* 관련 페이지 : [Spring 3.1 MVC Example](http://sleeplessinslc.blogspot.kr/2012/01/spring-31-mvc-example.html)
    
[Spring JavaConfig](http://static.springsource.org/spring/docs/3.1.x/spring-framework-reference/html/beans.html#beans-java) 은 순수자바(pure-java)를 이용한 스프링 컨테이너 설정을 의미한다. 다시 말해서, 가급적으로 XML을 사용하지 않는다. 이전에 XML에서 표현했던 방식을 자바 5 이상에서 제네릭Generic과 애노테이션Annotation에 의존하여 구현한다.

### JavaConfig의 이로움들

1. 주입Injection, 상속, 다형성 등의 기능을 수행한다.
2. 빈들beans에 대한 생성과 초기화에 대한 모든 권한을 가진다.
3. IDE와 같은 툴의 도움 없이도 리팩토링을 손쉽게할 수 있다.
4. 컨테이너container 초기화를 위해 많은 비용이 드는 클래스패스Classpath 스캐닝Scanning을 감소시킬 수 있는 능력
5. 필요에 따라 XML 혹은 프로퍼티property 등의 연결을 지원한다. 

이 블로그에 있는 대부분의 동작하는 예제와 코드들은 다운로드와 실행이 가능하다. 아래의 코드는 Spring MVC에 익숙한 사람이 본다는 가정을 하고 기본적인 것들은 살펴보지 않겠습니다. 이 블로그에 있는 예제들은 다음과 같은 목적으로 작성되었습니다.

### 예제의 작성목적
1. 스프링 XML을 사용하지 않고 Spring JavaConfig를 통해 다른 층을 연결할 수 있음을 설명한다.
2. 트랜잭션 관리를 설명한다.
3. [JSR-303](http://beanvalidation.org/1.0/spec/)을 이용한 빈에 대한 검증Bean validation을 설명한다.
4. 애플리케이션에 의한 RESTful Web Service 와 같은 REST 기능feature를 설명한다.
5. 웹서비스의 통합테스트integration-test에 RestTemplate로 사용됨을 설명한다.

간단히 말하면, 정말 긴 포스트라는... ;-)

이 블로그에서 참조하는 앱은 검색 및 발권이 가능한 비행 예약 시스템이다. 웹 애플리케이션은 웹, 서비스와 데이터 접근의 3계층으로 구분되어 있다. 각 계층은 최고 레벨 설정 혹은 WebConfig에서 시작된 계층적인 설정을 따르는 자바 설정Java Configuration을 가진다. 
![shown below with the WebConfig](http://1.bp.blogspot.com/-J8d58r3EfW8/Twe--sFJtiI/AAAAAAAABBA/nV8AxpWN3QU/s640/dependencies.png)

유닛 테스트를 용이하게 할 수 있도록, 설정 클래스(the Config classes)들은 인터페이스/구현체interface/imple로 분리했다.

### 데이터 접근 계층의 Java Config 모듈은 다음과 같다.
``` java
@Configuration
@EnableTransactionManagement
public class DefaultDaoConfig implements DaoConfig, TransactionManagementConfigurer {
  
  // Dao Initialization
  @Bean
  public FlightDao getFlightDao() {
     return new FlightDaoImpl(getSessionFactory());
  }
   ...other DAOs

  // Session Factory configuration for Hibernate
  @Bean
  public SessionFactory getSessionFactory() {
    return new AnnotationConfiguration().addAnnotatedClass(Ticket.class)
      .addAnnotatedClass(Flight.class).addAnnotatedClass(Airport.class)
      .configure().buildSessionFactory();
  }

  // Transaction manager being used
  @Override
  public PlatformTransactionManager annotationDrivenTransactionManager() {
    return new HibernateTransactionManager(getSessionFactory());
  }
}
```

앞에서 나온 [@EnableTransactionManagement](http://static.springsource.org/spring/docs/3.1.x/javadoc-api/org/springframework/transaction/annotation/EnableTransactionManagement.html) 애노테이션의 이득은 @Transactional 가 달린 서비스들에 대해서 스프링의 애노테이션 기반 트랜잭션 관리 기능들을 사용할 수 있다는 것이다. [@Transactional](http://static.springsource.org/spring/docs/3.1.x/javadoc-api/) 애노테이션은 스프링 컨테이너에서 메소드를 위한 트랜잭션이 제공된다는 것을 알려준다. 서비스 클래스에 선언된 @Transaction도 유사하다. 또한 노트에 있는 FlightDao와 TicketDao 빈즈는 [@Autowired](http://static.springsource.org/spring/docs/3.1.x/javadoc-api/index.html?index-filesindex-1.html) 애노테이션을 사용하지 않고  생성자를 통해 명시적으로explicitly 정의되어 있다. @Autowired는 스프링 프레임워크에 의해 인자가 없는(no-args) 생성자를 가진 DAO들에 사용할 수 있다(@Autowired는 단 하나의 생성자에만 사용할 수 있다는 제한이 있다. 여러 개의 생성자에 @Autowired가 붙으면 어느 생성자를 이용해서 DI해야 할지 스프링이 알 수 없기 때이다). 
--@Autowired는 XML의 타입에 의한 자동와이어링 방식을 생성자 필드, 수정자 메소드, 일반 메소드의 네 가지로 확장한 것이다.  필드나 프로퍼티 타입을 이용해 후보 빈을 찾는다. @Resource는 이름으로 후보 빈을 찾는가?

``` java
@Transactional(readOnly = true)
public class ReservationServiceImpl implements ReservationService {
  private final FlightDao flightDao;
  private final TicketDao ticketDao;

  public ReservationServiceImpl(FlightDao flightDao, TicketDao ticketDao) {...}

  @Override
  @Transactional(rollbackFor = { NoSeatAvailableException.class }, readOnly=false)
  public Ticket bookFlight(Reservation booking) throws  NoSeatAvailableException {
   ...
  }

  @Override
  public List<Ticket> getReservations() {...}
}
```

서비스 계층은 애플리케이션 서비스들에 정의된 설정Config에 대응하게 된다.
``` java
@Configuration
public class DefaultServiceConfig implements ServiceConfig {
  @Autowired
  private DaoConfig daoConfig;

  @Bean
  @Override
  public AirlineService getAirlineService() {
    return new AirlineServiceImpl(daoConfig.getFlightDao(), daoConfig.getAirportDao());
  }
  .........other services ...
}

```

강한 의문이 드는 것이 하나있다. 리플렉션하지 않고 명확하게 테스트 목적에 맞는 가짜 DaoConfig를 auto-wired 하여 내놓을 수 있을까? 생각해보자, 스프링의 컨테이너가 no-arg 생성자를 기대하는 것처럼, Config들은 초기화시 생성자를 이용할 없다. 한가지 방법이 있다면 Config를 위해서 setter를 정의하듯 mock에도 같은 것을 적용할 수 있다. 다음에 구현된 ControllerConfig 처럼 말이다(읽고보니, 앞서 나온 @Autowired를 생성자가 없는 경우에는 setter에 정의를 해서 사용할 수 있다는 것 같은데... )

``` java
@Configuration
public class ControllerConfig {
  private ServiceConfig serviceConfig;
  
  @Autowired 
  public void setServiceConfig(ServiceConfig serviceConfig) {
    this.serviceConfig = serviceConfig;
  }
 
  @Bean
  public FlightsController getFlightController() {
    return new FlightsController(serviceConfig.getAirlineService(), serviceConfig.getAirportService());
  }
  ... other controllers...
}
```

FlightContoller는 다음과 같다.

``` java FlightContoller
@Controller
public class FlightsController {
  private final AirlineService airlineService;
  private final AirportService airportService;

  public FlightsController(AirlineService airlineService, AirportService airportService) {...}

  // Flight search page initialization
  @RequestMapping(value = "/searchFlights.html", method = RequestMethod.GET)
  public ModelAndView searchFlights() throws Exception {
    List<Airport> airports = airportService.getAirports();

    ModelAndView mav = new ModelAndView("searchFlights");

    mav.addObject("airports", airports);

    return mav;
  }

  // Searching for a flight. Note that the Critiera object is automatically bound with 
  // the corresponding form post parameters
  @RequestMapping(value = "/searchFlights.html", method = RequestMethod.POST)
  public ModelAndView searchFlights(FlightSearchCriteria criteria) throws Exception {

    ModelAndView mav = new ModelAndView("searchFlights");
     ....
    FlightSearchResults searchResult = airlineService.getFlights(criteria);

    mav.addObject("flightSearchResult", searchResult);

    return mav;
  }
}
```

빈에 있는 @Controller 애노테이션은 스프링에 있는 MVC 유사체들과 같은 표시를 하며 request url과 web request 제공 등에 대한 동일한 매핑이 된다. 콘트롤러들에 대하여 보충설명을 하자면,
1. 다른 특별한 기본 클래스에 대한 확장이 필요없다.
2. [@RequestMapping](http://static.springsource.org/spring/docs/3.1.x/javadoc-api/index.html?index-filesindex-1.html) 애노테이션은 HTTP method나 리소스 서비스에 정의된다. 예를 들면, "/searchFlight"에 대한 요청에서  GET과 POST 요청에 따라 다르게 반응하도록 되어 있는 것을 볼 수 있다.

이 모든 계층의 설정들은 WebConfig에서 함께 묶을 수도 있고, Spring MVC Servlet 설정시 요구하는 다른 컴포넌트들을 별도로 정의할 수도 있다. [@Import](http://static.springsource.org/spring/docs/3.1.x/javadoc-api/org/springframework/context/annotation/Import.html) 애노테이션은 스프링 컨테이너에게 Config에 포함되어 필요한 Config들을 표시한다. 혹은 web.xml에서 Servlcet 정의부분에서 같은 방법으로 표시할 수 있다.

``` java WebConfig
@EnableWebMvc
@Configuration
@Import({ ControllerConfig.class, DefaultServiceConfig.class, DefaultDaoConfig.class })
public class WebConfig extends WebMvcConfigurerAdapter 
// Validator for JSR 303 used across the application
  @Override
  public Validator getValidator() {...}

 // View Resolver..JSP, Freemarker etc
  @Bean
  public ViewResolver getViewResolver() {...}

 // Mapping handler for OXM
  @Bean
  public RequestMappingHandlerAdapter getHandlerAdapter() {.. }
 
 @Override
  public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
    configurer.enable();
  }
}

```

[@EnableWebMvc](http://static.springsource.org/spring/docs/3.1.x/javadoc-api/) 애노테이션은 이것은 MVC 애플리케이션이라고 스프링에게 설명하는데 필요하다. [WebMvcConfigurerAdapter](http://static.springsource.org/spring/docs/3.1.x/javadoc-api/) 확장하여 Spring Servlet의 변경에 대한 허용 경로 조정을 구현한다. 혹은 WebMvcConfigurationSupport를 바로 상속받거나 필요한 조작을 제공할 수도 있다.

### Validation of Requests using JSR-303 Bean Validation API(JSR-303 API를 이용한 요청에 대한 검증)
[JSR-303](http://jcp.org/en/jsr/detail?id=303)은 자바 플랫폼 검증에 대한 표준을 제공한다. 검증을 원할 때 사용하면 된다. API의 애노테이션은 검증을 강제하는 것을 명시하고 런타임시에도 같다. 하이버네이트Hibernate를 사용한 이 예제에서 추가된 Hibernate-Validator는 API를 구현한 것이며 이것은 자연스러운 선택이다. 이 예제에 있는 Reservation 클래스는 JSR 애노테이션에서 제공하는 실행시 검증 및 메시지를 제공한다. 원한다면, 메시지는 리소스 번들에서 구성할 수 있다.

``` java Reservation
public class Reservation {
  @XmlElement
  @NotBlank(message = "must not be blank")
  private String reservationName;

  @Min(1)
  @XmlElement
  private int quantity = 1;

  @XmlElement
  @NotNull(message = "Flight Id must be provided")
  private Long flightId;
  ....
}
```

웹 배포 설명(The web deployment descriptor)은 애플리케이션에 집중되어 있으며, [DispatchServlet](http://static.springsource.org/spring/docs/3.1.x/javadoc-api/index.html?index-filesindex-1.html)는 WebConfig의 위치 및 Context class에서 사용한다는 것을 정의하고 있다. 앞서 언급했듯이, WebConfig에서 임포트import했던 Config들을 web.xml 안에 Config는 콤마(,)로 구분된 목록으로 제공할 수 있다. 
``` xml web.xml
<web-app>
        <servlet>
                <servlet-name>springExample</servlet-name>
                <servlet-class>org.springframework.web.servlet.DispatcherServlet
                </servlet-class>
                <init-param>
                        <param-name>contextClass</param-name>
                        <param-value>org.springframework.web.context.support.AnnotationConfigWebApplicationContext
                        </param-value>
                </init-param>
                <!-- Tell Spring where to find the Config Class
                     This is the hook in to the application. -->
                <init-param>
                        <param-name>contextConfigLocation</param-name>
                        <param-value>com.welflex.web.WebConfig</param-value>
                </init-param>
                <load-on-startup>1</load-on-startup>
        </servlet>
        ....
</web-app>
```

### 단위테스트Unit Testing
단위 테스트 계층은 [Spring Testing Framework](http://static.springsource.org/spring/docs/current/spring-framework-reference/html/testing.html)를 통해 쉬워진다. 예를 들어 서비스 계층을 테스트 할 때, 데이터베이스에 대한 연계가 필요 없으며 양방향에 대한 반응을 모조할 수 있다. 이런 이유로, DAO config 객체들은 보다시피 모조체를 제공한다.

``` java MockDaoConig
@Configuration
@EnableTransactionManagement
public class MockDaoConfig implements DaoConfig {
 
  @Bean
  public FlightDao getFlightDao() {
    return Mockito.mock(FlightDao.class);
  }
  ..// Other mocks
}

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {MockDaoConfig.class,
    DefaultServiceConfig.class }, loader = AnnotationConfigContextLoader.class)
public class AirportServiceTest {
  @Autowired
  private AirportDao airportDaoMock;

  @Autowired
  private ApplicationContext ctx;

  @Test
  public void getAirportCode() {
    // Obtain Airport service from the context
    AirportService a = ctx.getBean(AirportService.class);

    // Alternatively, instatiate an Airport Service by providing a mock
    AirportService a2 = new AirportServiceImpl(airportDaoMock);
  }
  ...
}
```

테스트에 앞서, Config는 ServiceConfig의 구현체와 DaoConfig의 모조체를 구현하는데 사용된다. 이를 통해 모조 데이터 접근 계층을 이용하여 서비스 계층을 테스트할 수 있게 된다. 몇몇 사람들은 컨테이너에 의해 모든 의존성이 연계되어 제공되는 클래스를 선호하는 것에 비해, 필요로 하는 의존성을 명확하게 연계wiring에 시킨 각각의 클래스들을 격리시켜서 테스트하는 것을 선호한다. 테스트 유형에 대해서는 다른 블로그들에서 고론되지만 명확하게 어떤 유형이 적합하다고 결론짓기는 어렵다. 테스트를 위해서 모든 빈들의 모조품이 필요한 것은 아니고 필요하면 별도의 Config 객체를 선언할 수 있다. 서비스 계층에서 사용되는 데이터 접근 모조체처럼, 컨트롤러 계층은 테스트를 위해 서비스 계층 모조 객체들을 사용할 수 있다.

### Spring MVC RESTful Services

이 예제의 목표중 하나는 웹 애플리케이션이 서비스로서도 좋다는 것을 설명하는 것이다. 컨트롤러를 따르면서 웹 애플리케이션을 대표함과 서비스 사용자a service consumer임을 알 수 있다.

``` java ReservationsController
@Controller
public class ReservationsController {
  private final AirlineService airlineService;
  private final ReservationService reservationService;

  public ReservationsController(ReservationService reservationService, AirlineService airlineService) {...}

  @RequestMapping("/reservations.html")
  public ModelAndView getReservationsHtml() {...}

  @RequestMapping(value = "/reservations/{reservationId}.html", method = RequestMethod.GET)
  public ModelAndView getReservationHtml(@PathVariable Long reservationId) {...}

  // Web Service Support to provide XML or JSON based of the Accept Header attribute
  @RequestMapping(value = "/reservations", method = RequestMethod.GET, produces = {
      MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE })
  @ResponseBody
  public Reservations getReservations() {...}

  @RequestMapping(value = "/reservations/{reservationId}", method = RequestMethod.GET, produces = {
      MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE })
  @ResponseBody
  public Ticket getReservation(@PathVariable Long reservationId) {...}
  ....
}
```

앞서 본 컨트롤러에서, getReservationHtml() 메소드는 getReservations() 에서 반환하는 XML 혹은 JSON 으로 구성된 예약정보를 HTML으로 구성하여 반환한다. 반환체의 유형은 서비스에서 제공하는 Accept header에 의존한다. @ResponseBody 애노테이션은 결과로 반환되는 객체가 즉각적으로 view를 대상으로 하지 않는다는 것을 의미한다(@ResponseBody에서 produces로 정의한 형태로 변환되어 반한된다는 것을 뜻한달까?). 예를 들면, XML을 마샬링 언마샬링하기 위해 [Spring OXM](http://static.springsource.org/spring-ws/site/reference/html/oxm.html)을 사용했다(via JAXB). [RestTemplate](http://static.springsource.org/spring/docs/3.1.x/javadoc-api/index.html?index-filesindex-1.html)를 이용한 컨트롤러의 통합테스트는 다음과 같다. 

``` java WebServiceIntegrationTest
public class WebServiceIntegrationTest {
  private RestTemplate template;
  private static final String BASE_URL = "http://localhost:9091";

  @Before
  public void setUp() {
    // Create Rest Template with the different converters
    template = new RestTemplate();
    List<HttpMessageConverter<?>> converters = ...;
    template.setMessageConverters(converters);
  }

  private static final String FLIGHT_NUMBER = "LH 235";

  @Test
  public void reservations() {
    Flight singleFlight = template.getForObject(BASE_URL + "/flights/" + FLIGHT_NUMBER,
      Flight.class);

    Reservation res = new Reservation();
    res.setFlightId(singleFlight.getId());
    res.setQuantity(10);
    res.setReservationName("Sanjay Acharya")

   // Post to create a ticket
    Ticket ticket = template.postForObject(BASE_URL + "/bookFlight", res, Ticket.class);
    assertNotNull(ticket);
    System.out.println("Ticket reserved:" + ticket);

    // All Reservations
    Reservations reservations = template.getForObject(BASE_URL  + "/reservations", Reservations.class);
    assertTrue(reservations.getTickets().size() == 1);

    // Single reservation
    assertNotNull(template.getForObject(BASE_URL + "/reservations/" + ticket.getId(), Ticket.class));
  }
}
```

RestTemplate에 대한 보다 자세한 정보는 [Rest Client Framework](http://sleeplessinslc.blogspot.com/2010/02/rest-client-frameworks-your-options.html) 를 확인하라.

### 참고사항Side Note
이 예제들에서 나는 내 POJO들에서 [equals(), hashCode() 와 toString()](http://pojomatic.sourceforge.net/pojomatic/index.html) 를 구현하는 것을 용이하게 하기 위해서 [Pojomatic](http://pojomatic.sourceforge.net/)이라고 불리는 프레임워트를 사용했다. 내가 이전에 작성했던 글에서 사용한 Apache Commons project에서 제공하는 툴보다 간결하여 좋아한다. 결과는 같다(Check the same out, 동작방식은 같다는 의미 같은데...).

* 추가 :  제 13회 한국스프링 사용자 모임 세미나에서 백기선님이 발표한 내용을 참조하면 좋을 듯 하다.
    * Spring MVC를 손쉽게 테스트하기 (백기선) : [동영상](http://www.youtube.com/playlist?list=PLHETc3O85eXKPqxRkx17tHjEmIwUWlw1d&feature=view_all) , [발표 자료](http://seminar.ksug.org/resources/pubs/ksug-spring-test-3.2.pdf)

### 예제실행Running Example
Spring MVC 예제에 Flight를 표현한 예제를 동봉한다. 이것은 실행될 때 메모리 데이터베이스를 이용하여 중요한 데이터들을 제공한다. The model package in the example is overloaded as far as its responsibilities go(요건 무슨 뜻인지 말 모르겠다). POJO들은 데이터베이스 모델 객체와 데이터 전송 객체(DTO, Data Transfer Object)로 활용된다. 예제의 단위 테스트 혹은 통합테스트는  결코by no means 광범위한 설계나 목적을 증명하기 위한 용도로 사용되었다.

1. [Download the example from Here](http://home.comcast.net/~acharya.s/java/springMvcExample.zip)
2. mvn jetty:run 실행, 애플리케이션 시작
3. mvn integration:test 실행, Web Service 확인
4. http://localhost:8080 에서 Flight Web application 확인할 수 있다.
5. 직접 접근 가능한 Web Service url
    * [http://localhost:8080/reservations](http://localhost:8080/reservations)
    * [http://localhost:8080/reservations/{id}](http://localhost:8080/reservations/{id})
    * [http://locahost:8080/flights](http://locahost:8080/flights)
    * [http://localhost:8080/airports](http://localhost:8080/airports)

### 결론Conclusion
나는 Spring Java Config를 이용한 와이어링과 객체 초기화에 대한 편리함을 찾아냈다. 의존성 트리의 동작은 매우 직관적이다. 추가적으로, Strong Typing을 통해, Spring IDE와 같은 툴 없이도 리팩토링을 할 수 있다. 내가 다른 웹 프레임워크보다 Spring MVC를 좋아하는 이유는 간결한 Front Controller/Dispatcher Pattern 때문이다. Ajaxian technology(Ajax 기술)처럼 사용자 경험을  강화할 수 있다. 매우 비간섭적인(non-invasive) 프레임워크다. 더군다나 스프링을 사용할 때는 모든 계층을 관통할 필요가 없다. RESTful 은 Ajax 기반 기술을 바탕으로 상호작용과 시스템 기반의 웹 애플리케이션 개발을 용이하게 한다. 애플리케이션을 상품화할 때 Spring MVC를 사용할 수 없다면 애석할 것이다. If anyone who does have experience with the same stumbles upon this BLOG, I would love to hear of the same.

같은 MVC Application 예제를 활용한, Twitter Bootstrap을 사용하고 Spring Security를 추가한, [Spring Security Stateless Cookie Authentication BLOG](http://sleeplessinslc.blogspot.com/2012/02/spring-security-stateless-cookie-based.html)을 봐라.

추가 - 간단하게 설명과 코드를 업로드 했다.  [Spring MVC 3.1 Presentation/Tutorial](http://sleeplessinslc.blogspot.com/2012/07/spring-mvc-31-presentation-and-tutorial.html) 를 보면된다.


### 참고사항
* [스프링 3.1 (1) JavaConfig의 위험성](http://toby.epril.com/?p=1168)