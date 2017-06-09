# invesdwin-webproxy
A massively parallel download manager for web scraping that supports proxy servers. Developed as modules for the [invesdwin-context](https://github.com/subes/invesdwin-context) module system. 

## Maven

Releases and snapshots are deployed to this maven repository:
```
http://invesdwin.de/artifactory/invesdwin-oss-remote
```

Dependency declaration:
```xml
<dependency>
	<groupId>de.invesdwin</groupId>
	<artifactId>invesdwin-webproxy</artifactId>
	<version>1.0.0-SNAPSHOT</version>
</dependency>
```
## Legal Discussion



## Architecture

The following context diagram shows what components this project consists of:

<p align="center"><img src="https://github.com/subes/invesdwin-webproxy/raw/master/invesdwin-webproxy-parent/invesdwin-webproxy/doc/webproxy_context.png" alt="Context Diagram" width="60%" /></p>

- **webproxy**: This is the actual `invesdwin-webproxy` module you use in your application to handle the downloads. Just inject the `IWebproxyService` into your spring beans and call the following provided methods:
	- `getString(GetStringConfig, URIs)`: to download via [HttpClient](http://hc.apache.org/httpclient-3.x/). You could parse the HTML string with [JSoup](https://jsoup.org/) to extract the required data or directly process a REST service result as CSV/XML/JSON. By giving multiple requests at the same time here, they will be processes in parallel for maximum performance. The `GetStringConfig` allows to configure things like browser agent, parallelity, proxy pool settings, proxy quality, retries, visited URI filtering and callbacks. Since proxies often are restricted or give garbage results, the `AProxyResponseCallback` can be provided with an implementation that verifies the payload of a proxy response against an expected result. For example you could check for a title string that should be in the given returned web site via `HtmlPageTitleProxyResponseCallback` and/or you could request the website to be downloaded twice with the same result by more than one proxy server via `DownloadTwiceProxyResponseCallback`. You can also provide a different kind of callback with `AStatisticsCallback` that allows you to measure statistics of the downloads. For example the `ConsoleReportStatisticsCallback` will print a nice summary to the console when calling its `logFinalReport()` method. You can even reuse the same callback instance over multiple requests to measure aggregated statistics.
	- `getPage(GetPageConfig, URIs)`: to download via [HtmlUnit](http://htmlunit.sourceforge.net/) which provides an in-memory/headless web browser that supports Javascript and CSS for more elaborate and dynamic parsing needs. Try the `page.asText()` method to parse the result as a TEXT/CSV file to get more robust code that does not rely on the actual HTML tags. By giving multiple requests at the same time here, they will be processes in parallel for maximum performance.  The `GetPageConfig` allows configuration beyond what `GetStringConfig` allows to configure a threshold for page refresh (might be a redirect), to enable CSS and to enable Javascript. You can also provide an implementation of `JavascriptWaitCallback` to define how long the download should wait for Ajax updates for data loading or initial rendering to finish when working with complex websites.
	- `newWebClient(GetPageConfig)`: if you want to manage an HtmlUnit session over multiple page requests manually. This will give you an instance of a preconfigured `WebClient` instance that you can use yourself. You have to manage parallelization here yourself, though it is not recommended to reuse the same `WebClient` instance in multiple threads, instead make sure to get a new instance for each thread. Also make sure to shut down the `WebClient` instance by calling `closeAllWindows()` after you are done with it.
	- `newProxy(GetStringConfig)`: if you want to fetch a specific proxy from the available pool by applying advanced filtering like timezone/country/quality/type and then setting it as a fixed proxy in `GetStringConfig`/`GetPageConfig`. The proxies don't have to be returned, since they will just rotate automatically. Proxies will automatically be verified and warmed up to make sure they actually work before they are returned from the pool, this happens here in the same way as it is done internally by the `getString`/`getPage` methods. Though if a proxy does not work you have to manually call `IBrokerService.addToBeVerifiedProxies` so the broker can update this information in its database and schedule a reverification of the proxy server (this normally happens automatically when using the other methods).
	- Advanced Configuration: the following system properties can be used to apply advanced customization:
```properties
# defines how many downloads are possible in parallel
de.invesdwin.webproxy.WebproxyProperties.MAX_PARALLEL_DOWNLOADS=100
# sleep time that is given for automatic page refreshes to occur
de.invesdwin.webproxy.WebproxyProperties.PROXY_VERIFICATION_REDIRECT_SLEEP=15 SECONDS
# may increase the number of detected proxies at the cost of performance. If deactivated only invalid responses may cause a retry
de.invesdwin.webproxy.WebproxyProperties.PROXY_VERIFICATION_RETRY_ON_ALL_EXCEPTIONS=false
# how long a proxy may reside in the pool before it is being reverified
de.invesdwin.webproxy.WebproxyProperties.PROXY_POOL_WARMUP_TIMEOUT=10 MINUTES
# if proxies should have a pause after a download, before the next download is started
de.invesdwin.webproxy.WebproxyProperties.PROXY_POOL_COOLDOWN_ALLOWED=true
# determines the randomized minimum limit of the proxy pause after a download
de.invesdwin.webproxy.WebproxyProperties.PROXY_POOL_COOLDOWN_MIN_TIMEOUT=100 MILLISECONDS
# determines the randomized maximum limit of the proxy pause after a download
de.invesdwin.webproxy.WebproxyProperties.PROXY_POOL_COOLDOWN_MAX_TIMEOUT=15 SECONDS
# how long a download may take maximally. This may help against evil or slow proxies that don't cause a timeout but also don't return anything
de.invesdwin.webproxy.WebproxyProperties.DEFAULT_MAX_DOWNLOAD_TRY_DURATION=10 MINUTES
# how many retries are allowed normally (0-99). Proxy caused tries do not count here
de.invesdwin.webproxy.WebproxyProperties.DEFAULT_MAX_DOWNLOAD_RETRIES=3
# determines if download exceptions should only cause a warning or should be rethrown. This is useful for debugging.
de.invesdwin.webproxy.WebproxyProperties.DEFAULT_MAX_DOWNLOAD_RETRIES_WARNING_ONLY=false
# all retries count here. This is a safety net if IsNotProxiesFaultProxyResponseCallback has bad rules.
de.invesdwin.webproxy.WebproxyProperties.MAX_ABSOLUTE_DOWNLOAD_RETRIES=100
# if not working proxies should be transmitted as such to the webproxy broker during the verification of proxies
de.invesdwin.webproxy.WebproxyProperties.AUTO_NOTIFY_ABOUT_NOT_WORKING_POOLED_PROXIES=false
```
- **registry**: specifically this is the `invesdwin-context-integration-ws-registry` module as provided by [invesdwin-context-integration](https://github.com/subes/invesdwin-context-integration) to act as a mediator for looking up the `webproxy-broker` instance to fetch working proxies from. If you do not enable proxy support in the `invesdwin-webproxy` module, then you don't have to actually deploy a registry server. You could even implement `IBrokerService` yourself and serve it as a spring bean to use an entirely different implementation. If using the standard implementation, then communication happen via a SOAP web service that is provided by the broker instance.
- **broker**: The `invesdwin-webproxy-broker` module mediates between the webproxy clients and the backend modules for proxy acquisition and database maintenance for keeping the information up to date about which proxies are working and what metadata (timezone, country, quality) they have. It schedules tasks for the crawler modules.
- **crawler**: The `invesdwin-webproxy-crawler` module provides a worker instance that you can host on multiple servers in order to distribute the workload of proxy acquisition and verification. When the broker requests new proxies, then the crawler can download raw proxy lists from your given `IProxyCrawlerSource` implementations. A raw proxy can also only contain an IP, in which case the port will be tried to be discovered automatically by checking the most common proxy ports. The `webproxy-crawler-tests` module provides some sample implementations, though they are included only for testing purposes here and should not be relied upon without asking the appropriate provider for permission if that is questionable. Though there are a few websites that specifically state that the given information is free for personal use. As an alternative you could buy a proxy subscription somewhere and parse the provided CSV file here. If you want to scan your local network for open proxies, you could enable random proxy discovery by setting the system property `de.invesdwin.webproxy.crawler.CrawlerProperties.RANDOM_SCAN_ALLOWED=true` which will scan random IPs on the internet and check the most common proxy ports for discovering public proxies. Though be aware that it is not a good idea to use this to acquire proxies from the internet as it is quite inefficient (since lots of port scan TCP packets might slow down your network if it is not suitable for this; also the implementation might not as efficient as it could be) and legally questionable depending on your country (though in most countries it is legal to discover and use public proxy servers to our knowledge, though we are no lawyers and can only advice against doing something like this). Multiple instances of the crawler will themselves ask the broker instance via the web service about the tasks they should perform. So the crawler instances don't have to be actually known by the broker server and it suffices to have the broker in the registry.
- **portscan**: The `invesdwin-webproxy-portscan` module needs to be deployed next to your crawler instance on any given installation. It is separated from the crawler process (which should be running with restricted permission on the operating system) because it requires root permission on the operating system to send TCP packets for the SYN stealth scan (as described by the [Nmap documentation](https://nmap.org/book/man-port-scanning-techniques.html)). Internally the [pcap](https://en.wikipedia.org/wiki/Pcap) library is used via the java binding of [jpcapng](https://sourceforge.net/projects/jpcapng/).
- **geolocation**: Another less important web service that is not included in the context diagram is the `invesdwin-webproxy-geolocation` module that uses [GeoIP](https://github.com/maxmind/geoip-api-java) and [GeoNames](http://www.geonames.org/) databases to determine the location, country and timezone for a given proxy IP. You can also use this service to check any other IP (e.g. for customers in your web shop) to determine where it comes from. You could also buy a premium GeoIP subscription to increase the accuracy of the resulting coordinates that are measured against locations provided by GeoNames.
