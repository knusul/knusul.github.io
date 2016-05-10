---
layout: post
title: "Consuming Google AdWords Api with Akka Spray"
date: 2016-05-02 19:45:22 +0200
comments: true
categories:  [spray, scala, adwords, api]
---

Google AdWords Api is a very tricky beast when under heavy use.
When you send too many requests per minute you get banned for 30s. Moreover the amount of queries that you cam perform varies durning the day.
I found consuming Google AdWords Api a perfect use case for Scala Akka and Spray frameworks.
Please take into account I'm beginer Scala programmer and it's one of my very first projects in that language.
We will build Spray Api that will accept json list of keywords, adWords locationId and languageId and we return the keywords enchanced with targetingIdea and trafficEstiamtor service data.
We use throttle actor(http://doc.akka.io/docs/akka/snapshot/contrib/throttle.html) to limit amount of queries to adWords Api. 

1. Start new Spray project like described at: `https://github.com/spray/spray-template/tree/on_spray-can_1.3`
```
 git clone git://github.com/spray/spray-template.git adwords-project
```

2. Add the following libraries to build.sbt

```
     "com.typesafe.akka" %% "akka-contrib" % akkaV,
     "io.spray" %%  "spray-json" % "1.3.2",
     "com.google.api-ads" % "ads-lib" % "2.11.0" exclude("commons-beanutils", "commons-beanutils"),
     "com.google.api-ads" % "adwords-axis" % "2.11.0" exclude("commons-beanutils", "commons-beanutils"),
     "commons-beanutils" % "commons-beanutils" % "1.9.2",
     "com.google.http-client" % "google-http-client-gson" % "1.21.0",
     "com.google.inject" % "guice" % "4.0",
     "org.slf4j" % "slf4j-simple" % "1.7.5",
     "javax.activation"  % "activation"  % "1.1.1"

```
3. Write JSON converter
We want to return json, and we need to convert adWords api response to json
In order to do that we write json converter:
```
implicit object AnyJsonFormat extends JsonFormat[Any] {
    def write(x: Any): JsValue = x match {
      case n: Int => JsNumber(n)
      case l: Long => JsNumber(l)
      case d: Double => JsNumber(d)
      case f: Float => JsNumber(f.toDouble)
      case s: String => JsString(s)
      case x: Seq[_] => seqFormat[Any].write(x)
      case m: Map[_, _] if m.isEmpty => JsObject(Map[String, JsValue]())
      // Get the type of map keys from the first key, translate the rest the same way
      case m: Map[_, _] => m.keys.head match {
        case sym: Symbol =>
          val map = m.asInstanceOf[Map[Symbol, _]]
          val pairs = map.map { case (sym, v) => (sym.name -> write(v)) }
          JsObject(pairs)
        case s: String => mapFormat[String, Any].write(m.asInstanceOf[Map[String, Any]])
        case a: Any =>
          val map = m.asInstanceOf[Map[Any, _]]
          val pairs = map.map { case (sym, v) => (sym.toString -> write(v)) }
          JsObject(pairs)
      }
      case a: Array[_] => seqFormat[Any].write(a.toSeq)
      case true        => JsTrue
      case false       => JsFalse
      case p: Product  => seqFormat[Any].write(p.productIterator.toSeq)
      case null        => JsNull
      case x           => JsString(x.toString)
    }

    def read(value: JsValue) = value match {
      case JsNumber(n) => n.intValue()
      case JsString(s) => s
      case JsTrue      => true
      case JsFalse     => false
    }
  }
```
We add the code above the routes into HttpService Trait
4. Add a route
Our only action will accept list of keywords and respond with enchanced data from targetingIdeaService
```
post {
    requestInstance { req =>
      val request = req.entity.asString.parseJson.convertTo[KeywordsRequest]
      complete {
        AdwordsCaller.handler(request.keywords, request.location_id, request.language_id).mapTo[Map[String, Map[String, Any]]]
      }
    }
  }
```
5. Implement an action
Adwords Caller implements Akka actor system that throttles requests to google limited to specific timeframe
```
case class CallAdwords(keywords: Array[String], location_id: Integer, langauge_id: Integer, session: AdWordsSession, AdWordsServices: AdWordsServices)
case object CallAdwords

class AdwordsCaller extends Actor {
  def receive = {
    case CallAdwords(keywords, location_id, language_id, session, adWordsServices) => sender ! GetAdwordsStats.run(keywords, location_id, language_id, session, adWordsServices)
  }
}

object AdwordsCaller {

  val system = ActorSystem()
  import system.dispatcher
  implicit val timeout = Timeout(60.second)

  val adwordsCaller = system.actorOf(Props[AdwordsCaller])
  val throttler = system.actorOf(Props(classOf[TimerBasedThrottler], 200 msgsPer 60.seconds))
  throttler ! SetTarget(Some(adwordsCaller))

  def handler(keywords: Array[String], location_id: Integer, language_id: Integer): Future[Any] = {
    val result = (throttler ? CallAdwords(keywords, location_id, language_id, null, null))
    return result
  }
}

```

6. Call AdWords Api
Finally GetAdwordsStats calls GoogleAdwords Api IdeaQuery and TrafficEstimation Query and combines results for each keyword

```
 def run(keywords: Array[String], location_id: Integer, language_id: Integer, session: AdWordsSession, adWordsServices: AdWordsServices): Map[String, Map[String, Any]] = {
      val oAuth2Credential = new OfflineCredentials.Builder().forApi(Api.ADWORDS)
    .fromFile("./ads.properties")
    .build()
    .generateCredential()
  val logger = LoggerFactory.getLogger(classOf[AdWordsServices])
  val session = new AdWordsSession.Builder().fromFile().withOAuth2Credential(oAuth2Credential)
    .build()
  val adWordsServices = new AdWordsServices()
    
    val ideaResults = runTargetingIdeaQuery(adWordsServices, session, keywords, location_id, language_id)
    val traficEstimatorResults = runTrafficEstimatorQuery(adWordsServices, session, keywords, location_id, language_id)
    var resultKeywords = Map[String, Map[String, Any]]()
    for (keyword <- keywords) {
      resultKeywords = resultKeywords + (keyword.toLowerCase() -> Map())
      for ((ideaKey, ideaValue) <- ideaResults(keyword.toLowerCase())) {
        var newVal = resultKeywords(keyword.toLowerCase()) + (ideaKey -> ideaValue)
        resultKeywords = resultKeywords + (keyword.toLowerCase() -> newVal)
      }
      for ((ideaKey, ideaValue) <- traficEstimatorResults(keyword)) {
        var newVal = resultKeywords(keyword.toLowerCase()) + (ideaKey -> ideaValue)
        resultKeywords = resultKeywords + (keyword.toLowerCase() -> newVal)
      }
    }

    return resultKeywords
  }

  def runTargetingIdeaQuery(adWordsServices: AdWordsServices, session: AdWordsSession, keywords: Array[String], location_id: Integer, language_id: Integer): Map[String, Map[String, Any]] = {

    val targetingIdeaService = adWordsServices.get(session, classOf[TargetingIdeaServiceInterface])
    val selector = new TargetingIdeaSelector()
    selector.setRequestType(RequestType.STATS)
    selector.setIdeaType(IdeaType.KEYWORD)

    selector.setRequestedAttributeTypes(Array(AttributeType.KEYWORD_TEXT, AttributeType.SEARCH_VOLUME,
      AttributeType.CATEGORY_PRODUCTS_AND_SERVICES, AttributeType.AVERAGE_CPC, AttributeType.COMPETITION))
    val paging = new Paging()
    paging.setStartIndex(0)
    paging.setNumberResults(2500)
    selector.setPaging(paging)
    val relatedToQuerySearchParameter = new RelatedToQuerySearchParameter()
    relatedToQuerySearchParameter.setQueries(keywords)

    val languageParameter = new LanguageSearchParameter()
    val language = new Language()
    language.setId(language_id.toLong)
    languageParameter.setLanguages(Array(language))
    val locationParameter = new LocationSearchParameter()
    val location = new Location()
    location.setId(location_id.toLong)
    locationParameter.setLocations(Array(location))

    selector.setSearchParameters(Array(relatedToQuerySearchParameter, languageParameter, locationParameter))
    val page = targetingIdeaService.get(selector)
    var resultKeywords = Map[String, Map[String, Any]]()
    if (page.getEntries != null && page.getEntries.length > 0) {

      for (targetingIdea <- page.getEntries) {
        val data = Maps.toMap(targetingIdea.getData)
        val keyword = data.get(AttributeType.KEYWORD_TEXT).asInstanceOf[StringAttribute]
        val categories = data.get(AttributeType.CATEGORY_PRODUCTS_AND_SERVICES).asInstanceOf[IntegerSetAttribute]
        var categoriesString = "(none)"
        if (categories != null && categories.getValue != null) {
          categoriesString = categories.getValue.toJson.prettyPrint
        }
        val averageMonthlySearches = data.get(AttributeType.SEARCH_VOLUME).asInstanceOf[LongAttribute]
          .getValue
        val cpc = data.get(AttributeType.AVERAGE_CPC).asInstanceOf[MoneyAttribute]

        val money = cpc.getValue
        var averageCpc = 0.0
        if (money != null) {
          averageCpc = money.getMicroAmount.toDouble / 1000000.0
        }
        var result: Map[String, Any] = Map("search_volume" -> averageMonthlySearches.toString(), "categories" -> categoriesString, "average_cpc" -> averageCpc.toString())
        resultKeywords = resultKeywords + (keyword.getValue() -> result)
      }
    } else {
      println("No related keywords were found.")
    }
    return resultKeywords
  }
  def runTrafficEstimatorQuery(adWordsServices: AdWordsServices, session: AdWordsSession, keywords: Array[String], location_id: Integer, language_id: Integer): Map[String, Map[String, Any]] = {

    val keywordEstimateRequests = new ArrayList[KeywordEstimateRequest]()

    for (keywordText <- keywords) {
      val keyword = new Keyword();
      keyword.setText(keywordText);
      keyword.setMatchType(KeywordMatchType.BROAD);
      val keywordEstimateRequest = new KeywordEstimateRequest();
      keywordEstimateRequest.setKeyword(keyword);
      keywordEstimateRequests.add(keywordEstimateRequest);
    }
    val adGroupEstimateRequests = new ArrayList[AdGroupEstimateRequest]()
    val adGroupEstimateRequest = new AdGroupEstimateRequest()
    adGroupEstimateRequest.setKeywordEstimateRequests(keywordEstimateRequests.toArray(Array()))
    adGroupEstimateRequest.setMaxCpc(new Money(null, 600000L))
    adGroupEstimateRequests.add(adGroupEstimateRequest)

    var campaignEstimateRequests = Array[CampaignEstimateRequest]()
    val campaignEstimateRequest = new CampaignEstimateRequest()
    campaignEstimateRequest.setAdGroupEstimateRequests(adGroupEstimateRequests.toArray(Array()))

    val location = new Location()
    location.setId(location_id.toLong)
    val language = new Language()
    language.setId(language_id.toLong)
    campaignEstimateRequests = campaignEstimateRequests :+ campaignEstimateRequest
    val selector = new TrafficEstimatorSelector()
    selector.setCampaignEstimateRequests(campaignEstimateRequests)

    val trafficEstimatorService = adWordsServices.get(session, classOf[TrafficEstimatorServiceInterface])
    val result = trafficEstimatorService.get(selector)

    var resultKeywords = Map[String, Map[String, Any]]()
    var index = 0
    for (
      campaignEstimate <- result.getCampaignEstimates; adGroupEstimate <- campaignEstimate.getAdGroupEstimates;
      keywordWithEstimate <- adGroupEstimate.getKeywordEstimates.zip(keywords)
    ) {
      var keyword = keywordWithEstimate._2
      var keywordEstimate = keywordWithEstimate._1
      val min = keywordEstimate.getMin
      val max = keywordEstimate.getMax
      val avg_total_cost = (max.getTotalCost.getMicroAmount + min.getTotalCost.getMicroAmount) / 2000000.0
      var result: Map[String, Any] = Map("avg_ctr" -> (min.getClickThroughRate + max.getClickThroughRate) / 2,
        "keyword" -> keyword, "avg_total_cost" -> avg_total_cost.toString(),
        "clicks_per_day" -> Array(min.getClicksPerDay, max.getClicksPerDay),
        "total_cost" -> Array(min.getTotalCost.getMicroAmount / 1000000, max.getTotalCost.getMicroAmount / 1000000),
        "average_position" -> Array(min.getAveragePosition, max.getAveragePosition))
      resultKeywords = resultKeywords + (keyword -> result)
    }
    return resultKeywords
  }
```
