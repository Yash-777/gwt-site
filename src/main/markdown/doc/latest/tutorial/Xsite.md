
<style>
code, .code {font-size: 9pt; font-family: Courier, Courier New, monospace; color:#007000;}
.highlight {background-color: #ffc;}
.strike {text-decoration:line-through; color:red;}
.header {margin-top: 1.5ex;}
.details {margin-top: 1ex;}
</style>

<p>
At this point, you've modified the initial implementation of the StockWatcher application, which simulated stock data in the client-side code. The current implementation now retrieves JSON-formatted data from your local server.
</p>
<p>
In this session, you'll make a call to a remote server instead. To do so you will have to work around SOP (Same Origin Policy) constraints.
</p>
<ol>
    <li><a href="#design">Review the requirements and design: access restrictions and asynchronous communication.</a></li>
    <li><a href="#server">Create a source of JSON data on a remote server.</a></li>
    <li><a href="#request">Request the data from the remote server.</a></li>
    <li><a href="#response">Handle the response.</a></li>
    <li><a href="#test">Test.</a></li>
</ol>

<p class="note" style="margin-left: 1.2em; margin-right: 1.5em;">
<b>Note:</b> For a broader guide to client-server communication in a GWT application, see <a href="../DevGuideServerCommunication.html">Communicate with a Server</a>.
</p>

<h2>Before you begin</h2>
<h3>The StockWatcher project</h3>
<p>
This tutorial builds on the GWT concepts and the StockWatcher application created in the <a href="gettingstarted.html">Build a Sample GWT Application</a> tutorial. If you have not completed the Build a Sample GWT Application tutorial and are familiar with basic GWT concepts, you can import the StockWatcher project as coded to this point.
</p>
<ol class="instructions">
    <li>
        <div class="header">Download the <a href="http://code.google.com/p/google-web-toolkit/downloads/detail?name=Tutorial-JSON-2.1.zip">StockWatcher project</a> (with JSON).</div>
    </li>
    <li>
        <div class="header">Unzip the file.</div>
    </li>
    <li>
        <div class="header">Import the project into Eclipse</div>
        <ol>
            <li>From the <code>File</code> menu, select the  <code>Import...</code> menu option.</li>
            <li>Select the import source General &gt; Existing Projects into Workspace. Click the <code>Next</code> button.</li>
            <li>For the root directory, browse to and select the StockWatcher directory (from the unzipped file). Click the <code>Finish</code> button.</li>
        </ol>
    </li>
</ol>
<p>
If you are using ant, edit the <code>gwt.sdk</code> property in StockWatcher/build.xml to point to where you unzipped GWT.
</p>

<p>
In order to actually run this tutorial, you will need either access to a server other than where StockWatcher is running, one that can run a PHP script, or be able to run a Python script on your machine.
</p>

<a name="design"></a>
<h2>1. Reviewing the requirements and design</h2>
<p>
As you modify the current implementation of StockWatcher to access data on a remote server, there are two issues to address:
</p>
<ul>
    <li>Access Restrictions: SOP (Same Origin Policy)</li>
    <li>Asynchronous Communication</li>
</ul>
<h3>Access Restrictions: Same Origin Policy</h3>
<p>
The Same Origin Policy (SOP) is a browser security measure that restricts client-side JavaScript code from interacting with resources <strong>not</strong> originating from the same domain name, protocol and port. The browser considers two pages to have the same origin only if these three values are the same. For example, if the StockWatcher application is running in a web page on http://abc.com:80 it cannot interact with stock data loaded from a different domain, http://xyz.com. It can't even load stock data from the same domain if the port is different, for example, http://abc.com:81.
</p>
<p>
The idea behind SOP is that, for security reasons, the browser should not trust content loaded from arbitrary websites. A malicious web page could inject code that steals data or otherwise compromises security.
</p>
<p>
As such, for accessing JSON-formatted stock data from another domain or port, the current implementation will not work. The web browser will block the HTTP call to retrieve the JSON.
</p>
<p class="note">For a more detailed description of SOP and its effect on GWT, read <a href="../FAQ_Server.html#What_is_the_Same_Origin_Policy,_and_how_does_it_affect_GWT?">What is the Same Origin Policy, and how does it affect GWT?</a></p>

<h4>Working around SOP</h4>
<p>
There are two options for working around SOP security:
</p>
<ul>
    <li>Proxy on your own server</li>
    <li>Load the JSON response into a &lt;script&gt; tag</li>
</ul>

<h5>Proxy on your own server</h5>
<p>
The first option is to play by the rules of SOP and create a proxy on your local server.
You can then make HTTP calls to your local server and have it go fetch the data from the remote server.
This works because the code running on your web server is not subject to SOP restrictions. Only the client-side code is.
</p>
<p>
Specifically for the StockWatcher application, you could implement this strategy by writing server-side code to download (and maybe cache) the JSON-encoded stock quotes from a remote server.
You could then use any mechanism we want for retrieving the data from the local server: GWT RPC or direct HTTP using RequestBuilder.
</p>
<p>
One downside to this approach is that it requires additional server-side code.
Another is that the extra HTTP call increases the latency of remote calls and adds to the workload on our web server.
</p>

<h5>Load the JSON response into a &lt;script&gt; tag</h5>
<p>
Another option is to dynamically load JavaScript into a &lt;script&gt; tag.
Client-side JavaScript can manipulate &lt;script&gt; tags, just like any other element in the HTML Document Object Model (DOM).
Client-side code can set the src attribute of a &lt;script&gt; tag to automatically download and execute new JavaScript into the page.
</p>
<p>
This strategy is not subject to SOP restrictions. So you can effectively use it to load JavaScript (and therefore JSON) from remote servers.
</p>
<p>
This is the strategy you'll use to get the JSON-formatted stock data from a remote server.
</p>

<h3>Asynchronous Communication</h3>
<p>
Dynamically loading the JavaScript into a &lt;script&gt; tag solves the SOP issue but introduces another.
When you use this method to load JavaScript, although the browser retrieves the code asynchronously, it doesn't notify you when it's finished.
Instead, it simply executes the new JavaScript.
However, by definition, JSON cannot contain executable code.
Put the two together and you'll realize that you can't load plain JSON data using a &lt;script&gt; tag.
</p>
<h4>JSON with Padding (JSONP)</h4>
<p>
To resolve the callback issue, you can specify the name of a callback function as an input argument of the call itself.
The web server will then wrap the JSON response in a call to that function.
This technique is called JSON with Padding (JSONP).
When the browser finishes downloading the new contents of the &lt;script&gt; tag, the callback function executes.
</p>
<pre class="code">
callback125([{"symbol":"DDD","price":10.610339195026,"change":0.053085447454327}]);
</pre>
<p>
Google Data APIs support this technique.
</p>
<p>
For StockWatcher, the additional requirement in the client-side code is that you include the name of the JavaScript function you're using as a callback in the HTTP request .
</p>

<h3>Implementation Strategies</h3>
<p>
Now that you understand the SOP issues surrounding cross-site requests, compare this implementation to the implementation for getting JSON data from a local server. You'll have to change some of the existing implementation but you'll be able to reuse some components as well. Most of the work will be in writing the new method, getJSON, which makes the call to the remote server.
</p>

<table>
    <tr>
        <th>Task</th>
        <th>Same-Site Implementation</th>
        <th>Cross-Site Implementation</th>
    </tr>
    <tr>
        <td>Making the call</td>
        <td>HTTP with Request Builder</td>
        <td>Embed a script whose src attribute is the URL of the JSON data with the name of the callback function appended.</td>
    </tr>
    <tr>
        <td>Server-side code</td>
        <td>Returns JSON string</td>
        <td>Returns a Javascript callback function with the JSON string</td>
    </tr>
    <tr>
        <td>Handling the response</td>
        <td>Use JavaScript eval() function to turn JSON string into JavaScript object</td>
        <td>Already a JavaScript object; cast it as a StockData array</td>
    </tr>
    <tr>
        <td>Data objects</td>
        <td>Create an overlay type: StockData</td>
        <td>Reuse the overlay type</td>
    </tr>
    <tr>
        <td>Handle Errors</td>
        <td>Create a Label widget to display error messages</td>
        <td>Reuse the Label widget</td>
    </tr>
</table>


<a name="server"></a>
<h2>2. Creating a data a source</h2>
<p>
In this tutorial, you have two options for setting up the stock data so that StockWatcher encounters SOP restrictions.
</p>
<ol>
    <li>If you have access to a server with PHP installed, you can use the PHP script below to generate the JSON-formatted stock data.</li>
    <li>If you don't have a server but have Python installed on your machine, you can use the Python script below to serve the stock data from a different port than StockWatcher is running on.</li>
</ol>
<h3>Actually using different server</h3>
<p>If you have access to a web server, then you can use the following PHP script to return the JSONP.</p>
<ol class="instructions">
    <li>
        <div class="header">Create a text file and name it <code>stockPrices.php</code></div>
        <div class="details"></div>
        <div class="details"><pre class="code">
&lt;?php

  header('Content-Type: text/javascript');
  header('Cache-Control: no-cache');
  header('Pragma: no-cache');

  define("MAX_PRICE", 100.0); // $100.00
  define("MAX_PRICE_CHANGE", 0.02); // +/- 2%

  $callback = trim($_GET['callback']);
  echo $callback;
  echo '([';

  $q = trim($_GET['q']);
  if ($q) {
    $symbols = explode(' ', $q);

    for ($i=0; $i&lt;count($symbols); $i++) {
      $price = lcg_value() * MAX_PRICE;
      $change = $price * MAX_PRICE_CHANGE * (lcg_value() * 2.0 - 1.0);

      echo '{';
      echo "\"symbol\":\"$symbols[$i]\",";
      echo "\"price\":$price,";
      echo "\"change\":$change";
      echo '}';

      if ($i &lt; (count($symbols) - 1)) {
        echo ',';
      }
    }
  }

  echo ']);';
?&gt;
</pre></div>
    </li>
    <li>
        <div class="header">Copy the PHP script to another server.</div>
    </li>
   <li>
        <div class="header">Open a browser and make a request for the JSON data.</div>
        <div class="details"><code>http://<i>[www.myStockServerDomain.com]</i>/stockPrices.php?q=ABC</code></div>
    </li>
   <li>
        <div class="header">The JSON string is returned.</div>
        <div class="details"><code>[{"symbol":"ABC","price":81.284083,"change":-0.007986}]</code></div>
        <div class="details">However, as you'll see in the next section, the StockWatcher application will not be able to make this request from its client-side code.</div>
    </li>
   <li>
        <div class="header">Make a request for the JSONP by appending the name of a callback function.</div>
        <div class="details"><code>http://<i>[www.myStockServerDomain.com]</i>/stockPrices.php?q=ABC&callback=callback125</code></div>
    </li>
   <li>
        <div class="header">The JSON is returned embedded in the callback function.</div>
        <div class="details"><code>callback125([{"symbol":"ABC","price":53.554212,"change":0.584011}]);
</code></div>
    </li>
</ol>


<h3>Simulating a second server</h3>
<p>
If you do not have access to a remote server, but have Python installed on your local machine, you can simulate a remote server. If you make an HTTP request to a different port, you will hit the same SOP restrictions as if you were trying to access a different domain.
</p>
<p>
Use the following script to serve data from a different port on your local machine.
For each stock symbol the Python script generates random price and change values in JSON format.
Notice in the BaseHTTPServer.HTTPServer constructor that it will be running on port 8000.
Also notice that the script supports the callback query string parameter.
</p>
<ol class="instructions">
    <li>
        <div class="header">Create a Python script and save it as <code>quoteServer.py</code></div>
        <div class="details"><pre class="code">
#!/usr/bin/env python2.4
#
# Copyright 2007 Google Inc. All Rights Reserved.

import BaseHTTPServer
import SimpleHTTPServer
import urllib
import random

MAX_PRICE = 100.0
MAX_PRICE_CHANGE = 0.02

class MyHandler(SimpleHTTPServer.SimpleHTTPRequestHandler):

  def do_GET(self):
    form = {}
    if self.path.find('?') &gt; -1:
      queryStr = self.path.split('?')[1]
      form = dict([queryParam.split('=') for queryParam in queryStr.split('&amp;')])

    body = '['

    if 'q' in form:
      quotes = []

      for symbol in urllib.unquote_plus(form['q']).split(' '):
        price = random.random() * MAX_PRICE
        change = price * MAX_PRICE_CHANGE * (random.random() * 2.0 - 1.0)
        quotes.append(('{"symbol":"%s","price":%f,"change":%f}'
                       % (symbol, price, change)))

      body += ','.join(quotes)

    body += ']'

<span class="highlight">    if 'callback' in form:
      body = ('%s(%s);' % (form['callback'], body))</span>

    self.send_response(200)
    self.send_header('Content-Type', 'text/javascript')
    self.send_header('Content-Length', len(body))
    self.send_header('Expires', '-1')
    self.send_header('Cache-Control', 'no-cache')
    self.send_header('Pragma', 'no-cache')
    self.end_headers()

    self.wfile.write(body)
    self.wfile.flush()
    self.connection.shutdown(1)

<span class="highlight">bhs = BaseHTTPServer.HTTPServer(('', 8000), MyHandler)</span>
bhs.serve_forever()
</pre></div>
    </li>
    <li>
        <div class="header">Save the script to the main StockWatcher directory.</div>
        <div class="details"></div>
    </li>

    <li>
        <div class="header">Make sure the Python interpreter is on your PATH.</div>
    </li>

    <li>
        <div class="header">Launch the script.</div>
        <div class="details">From the command shell, enter <code>python quoteServer.py</code></div>
        <div class="details">The server will start, although you won't see any output immediately. (It will log each HTTP request).</div>
    </li>
   <li>
        <div class="header">Open a browser and make a request for the JSON data.</div>
        <div class="details"><code>http://localhost:8000/?q=ABC</code></div>
    </li>
   <li>
        <div class="header">The JSON string is returned.</div>
        <div class="details"><code>[{"symbol":"ABC","price":81.284083,"change":-0.007986}]</code></div>
        <div class="details">However, as you'll see in the next section, the StockWatcher application will not be able to make this request from its client-side code.</div>
    </li>
   <li>
        <div class="header">Make a request for the JSONP by appending the name of a callback function.</div>
        <div class="details"><code>http://localhost:8000/?q=ABC&callback=callback125</code></div>
    </li>
   <li>
        <div class="header">The JSON is returned embedded in the callback function.</div>
        <div class="details"><code>callback125([{"symbol":"ABC","price":53.554212,"change":0.584011}]);
</code></div>
    </li>
</ol>

<a name="request"></a>
<h2>3. Requesting the data from the remote server</h2>
<p>
Now that you've verified that the server is returning stock data either as a JSON string or as JSONP, you can update StockWatcher to request and then handle the JSONP.

The RequestBuilder code is replaced by a call to the getJson method. The first parameter is an ID number that uniquely identifies each HTTP request.
</p>

<h3>Specifying the URL</h3>
<p>
Update the query URL so that it includes both the stock codes your are querying and the name of the callback function. Each callback function will have a unique ID number.
</p>
<h4>Update JSON_URL</h4>
<p>
This is the only difference that results in the implementation depending on whether the data is being served from a different domain or a different port.
</p>
<ol class="instructions">
    <li>
        <div class="header">In the StockWatcher class, change the JSON_URL constant as follows:
</div>
        <div class="details"></div>
        <div class="details">Change: <pre class="code">
  private static final String JSON_URL = GWT.getModuleBaseURL() + "stockPrices?q=";</pre></div>
        <div class="details">If your stock data is being served from a different port (the Python script), change JSON_URL to: <pre class="code">
  private static final String JSON_URL = "http://localhost:8000/?q=";</pre></div>
        <div class="details">If your stock data is being served from a different domain (the PHP script), specify the domain and full path to the stockPrices.php script: <pre class="code">
  private static final String JSON_URL = "http://<i>www.myStockServerDomain.com</i>/stockPrices.php?q=";</pre></div>
    </li>
    <li>
        <div class="header">Create a variable to hold the value of the callback ID.</div>
        <div class="details"><pre class="code">
  private Label errorMsgLabel = new Label();
<span class="highlight">  private int jsonRequestId = 0;</span></pre></div>
    </li>
     <li>
        <div class="header">Try to retrieve plain JSON data from the remote server.</div>
        <div class="details">Debug StockWatcher in development mode.</div>
        <div class="details">Enter a stock code.</div>
    </li>
     <li>
        <div class="header">StockWatcher displays the error message: Couldn't retrieve JSON.</div>
        <div class="details">To fix the SOP error, in the next step you'll pad the JSON with the callback function.</div>
    </li>

</ol>

<h3>Creating the callback method</h3>
<p>
In the same-site implementation, the JSON URL was appended with the stock codes and then a HTTP GET request sent to the server. RequestBuilder was used to construct the HTTP request.
</p>
<p>
In the cross-site implementation, the JSON URL is wrapped in the callback method. The RequestBuilder code is replaced by a JSNI function, getJSON(int, String, StockWatcher). The first parameter is an ID number that uniquely identifies each HTTP request.
</p>
<p>
The getJSON function creates a JavaScript script which makes the call to the server.
</p>

<h4>Update the refreshWatchList method</h4>
<ol class="instructions">
    <li>
        <div class="header"></div>
        <div class="details">Update the refreshWatchList method.</div>
        <div class="details"><pre class="code">
  /**
   * Generate random stock prices.
   */
  private void refreshWatchList() {
    if (stocks.size() == 0) {
      return;
    }

    String url = JSON_URL;

    // Append watch list stock symbols to query URL.
    Iterator&lt;String&gt; iter = stocks.iterator();
    while (iter.hasNext()) {
        url += iter.next();
        if (iter.hasNext()) {
            url += "+";
        }
    }

<span class="highlight">    // Append the name of the callback function to the JSON URL.
    url = URL.encode(url) + "&amp;callback=";

    // Send request to server by replacing RequestBuilder code with a call to a JSNI method.
    getJson(jsonRequestId++, url, this);</span>
  }
</pre></div>
</li>
    <li>
        <div class="header">Eclipse flags getJson.</div>
        <div class="details">Ignore the compile error; you'll write that method in a minute.</div>
    </li>
    <li>
        <div class="header">If you haven't already, delete the RequestBuilder code.</div>
        <div class="details">The RequestBuilder code is replaced by a call to the getJson method. So you no longer need the following code in the refreshWatchList method: </div>
        <div class="details"><pre class="code">
    // Send request to server and catch any errors.
    RequestBuilder builder = new RequestBuilder(RequestBuilder.GET, url);

    try {
      Request request = builder.sendRequest(null, new RequestCallback() {
        public void onError(Request request, Throwable exception) {
          displayError("Couldn't retrieve JSON");
        }

        public void onResponseReceived(Request request, Response response) {
          if (200 == response.getStatusCode()) {
            updateTable(asArrayOfStockData(response.getText()));
          } else {
            displayError("Couldn't retrieve JSON (" + response.getStatusText()
                + ")");
          }
        }
      });
    } catch (RequestException e) {
      displayError("Couldn't retrieve JSON");
    }
</pre></div>
    </li>

</ol>

<h3>Making the call to the remote server</h3>
<p>
What follows is the most important change in this implementation. This is where you work around the SOP restrictions by using the src attribute of the &lt;script&gt; tag to make the call to the remote server.
</p>
<p>
This JSNI method (getJSON) creates a dynamically-loaded &lt;script&gt; element. The src attribute is the URL of the JSON data with the name of a callback function appended. When the script executes, it fetches the padded JSON; the JSON data is passed as an argument of the callback function. When the callback function executes, it calls the Java handleJsonResponse method and passes it the JSON data as a JavaScript object.
</p>
<p>
Not only does this implementation include some embedded handwritten JavaScript (JSNI) but it uses a bridge method, a technique for calling back into the Java source code during development. (Remember, when you compile StockWatcher, all the client-side Java code is compiled into JavaScript.)
</p>
<h4>Implement the getJSON method</h4>
<ol class="instructions">
    <li>
        <div class="header">Add the getJson method to the StockWatcher class.</div>
        <div class="details"><pre class="code">
  /**
   * Make call to remote server.
   */
  public native static void getJson(int requestId, String url,
      StockWatcher handler) /*-{
   var callback = "callback" + requestId;

   // [1] Create a script element.
   var script = document.createElement("script");
   script.setAttribute("src", url+callback);
   script.setAttribute("type", "text/javascript");

   // [2] Define the callback function on the window object.
   window[callback] = function(jsonObj) {
   // [3]
     handler.@com.google.gwt.sample.stockwatcher.client.StockWatcher::handleJsonResponse(Lcom/google/gwt/core/client/JavaScriptObject;)(jsonObj);
     window[callback + "done"] = true;
   }

   // [4] JSON download has 1-second timeout.
   setTimeout(function() {
     if (!window[callback + "done"]) {
       handler.@com.google.gwt.sample.stockwatcher.client.StockWatcher::handleJsonResponse(Lcom/google/gwt/core/client/JavaScriptObject;)(null);
     }

     // [5] Cleanup. Remove script and callback elements.
     document.body.removeChild(script);
     delete window[callback];
     delete window[callback + "done"];
   }, 1000);

   // [6] Attach the script element to the document body.
   document.body.appendChild(script);
  }-*/;</pre></div>
    </li>
</ol>
<h4>Implementation Notes</h4>
<ul>
    <li>[1] The script starts by setting up a &lt;script&gt; element. The src attribute points to the URL that will retrieve the JSON data wrapped in a callback function.</li>
    <li>[2] The callback function is defined on the browser's window object. It receives as an argument a JavaScript object which is the JSON data returned by the server.</li>
    <li>[3] The callback function passes the JSON data as a JavaScript object to the Java method, handleJsonResponse.</li>
    <li>[4] A timeout function is defined to check for an unresponsive server or network problem; it checks a flag to see if the JSON callback was ever called.</li>
    <li>[5] Before the timeout function completes, it removes the new &lt;script&gt; element and the callback function from window.</li>
    <li>[6] Finally call appendChild() to attach the dynamically-loaded &lt;script&gt; element to the HTML document body. This causes the web browser to download the JavaScript referenced by the src attribute.</li>
</ul>
<h5>Handling multiple pending requests</h5>
<p>
This implementation generates callback function names sequentially in case of multiple pending requests. In particular, notice the syntax used to call the handleJsonResponse(JavaScriptObject) method:
</p>
<pre class="code">
handler.@com.google.gwt.sample.stockwatcher.client.StockWatcher::handleJsonResponse(Lcom/google/gwt/core/client/JavaScriptObject;)(jsonObj);
</pre>
<p>
You can see that once the JSON object is downloaded, the callback in the JSNI method is really just delegating to the Java method handleJsonResponse.
</p>

<h4>More about bridge methods</h4>
<p>
Calling Java methods from JavaScript is somewhat similar to calling Java methods from C code in JNI. In particular, JSNI borrows the JNI mangled method signature approach to distinguish among overloaded methods. JavaScript calls into Java methods are of the following form:
</p>
<pre class="code">
[instance-expr.]@class-name::method-name(param-signature)(arguments)
</pre>
<table>
    <tr>
        <th>Component</th>
        <th>Description</th>
        <th>Example</th>
    </tr>
    <tr>
        <td>[instance-expr.]</td>
        <td>Must be present when calling an instance method and must be absent when calling a static method</td>
        <td>handler.<br />(StockWatcher object instance)</td>
    </tr>
    <tr>
        <td>@class-name</td>
        <td>The fully-qualified name of the StockWatcher class</td>
        <td>@com.google.gwt.sample.stockwatcher.client.StockWatcher</td>
    </tr>
    <tr>
        <td>::method-name</td>
        <td>The name of the method we're calling</td>
        <td>::handleJsonResponse</td>
    </tr>
    <tr>
        <td>(param-signature)</td>
        <td>The handleJsonResponse method signature, defined with the JNI syntax</td>
        <td>(Lcom/google/gwt/core/client/JavaScriptObject;)</td>
    </tr>
    <tr>
        <td>(arguments)</td>
        <td>The jsonObj containing the JSON data</td>
        <td>(jsonObj)</td>
    </tr>
</table>
<p class="note">
For more information on manipulating Java objects from within the JavaScript implementation of a JSNI method, see the Developer's Guide, <a href="../DevGuideCodingBasics.html#DevGuideJavaScriptNativeInterface">Accessing Java Methods and Fields from JavaScript</a>.
</p>

<a name="response"></a>
<h2>4. Handling the response</h2>
<p>
At this point most of your work is done. The one difference is that the return value is already a JavaScript object, not a JSON string. Thus, in the asArrayOfStockData method you no longer  have to use the JavaScript eval() function to convert it.
</p>

<h4>Implement the handleJsonResponse method</h4>
<p>
If you receive a response from the server, then call the updateTable method to populate the Price and Change fields. You will still use the overlay type (StockData)  and the JsArray (asArrayOfStockData) that you wrote in the same-site implementation.
</p>
<p>
If a response does not come back from the server, you display a message. You can use the same displayError method and Label widget you created in the same-site implementation.
</p>

<ol class="instructions">
    <li>
        <div class="header">To the StockWatcher class, add the handleJsonResponse method.</div>
        <div class="details"></div>
        <div class="details"><pre class="code">
  /**
   * Handle the response to the request for stock data from a remote server.
   */
  public void handleJsonResponse(JavaScriptObject jso) {
    if (jso == null) {
      displayError("Couldn't retrieve JSON");
      return;
    }

    updateTable(asArrayOfStockData (jso));

  }
</pre></div>
        <div class="details">Eclipse flags JavaScriptObject.</div>
    </li>
    <li>
        <div class="header">Include the import declaration.</div>
        <div class="details"><pre class="code">
import com.google.gwt.core.client.JavaScriptObject;</pre></div>
        <div class="details">Eclipse flags asArrayOfStockData.</div>
        <div class="details">The asArrayOfStockData is expecting a String not a JavaScriptObject.</div>
    </li>
</ol>

<p>
<h4>Modify the arrayOfStockData method</h4>
<p>
In this implementation the response is a JavaScript object, not a string. Your next step is to modify the asArrayOfStockData method. Rather than convert the JSON string into a JavaScript array, it needs to cast the JavaScript object returned from the JSNI method as an array of StockData.
</p>
<ol class="instructions">
    <li>
        <div class="header">Modify the asArrayOfStockData method as follows:</div>
        <div class="details">Change:<pre class="code">
  /**
   * Convert the string of JSON into JavaScript object.
   */
  private final native JsArray&lt;StockData&gt; asArrayOfStockData(String json) /*-{
    return eval(json);
  }-*/;
</pre></div>

        <div class="details">To:<pre class="code">
  /**
   * Cast JavaScriptObject as JsArray of StockData.
   */
  private final native JsArray&lt;StockData&gt; asArrayOfStockData(JavaScriptObject jso) /*-{
    return jso;
  }-*/;
</pre></div>
    </li>
</ol>

<a name="test"></a>
<h2>5. Testing</h2>
<p>
Whether you chose to serve the JSON-formatted stock data from a different domain or a different port, the new StockWatcher implementation should work around any SOP access restrictions and be able to retrieve the stock data.
</p>
<h3>Test in development mode</h3>
<h4>Stock Data served from a different port</h4>
<ol class="instructions">
     <li>
        <div class="header">Make sure the Python server is running.</div>
        <div class="details">If it's not, at the command line, enter <code>python quoteServer.py</code></div>
    </li>
    <li>
        <div class="header">In the browser running in development mode, refresh StockWatcher.</div>
    </li>
     <li>
        <div class="header">Add a stock code.</div>
        <div class="details">StockWatcher displays the Price and Change data. The information is now coming from a different port.</div>
    </li>
     <li>
        <div class="header">Shutdown the Python server.</div>
        <div class="details">StockWatcher displays the error: Couldn't retrieve JSON</div>
    </li>
     <li>
        <div class="header">Restart the Python server.</div>
        <div class="details">StockWatcher clears the error and continues displaying Price and Change updates.</div>
    </li>
</ol>
<h4>Stock Data served from a different domain</h4>
<ol class="instructions">
    <li>
        <div class="header">In the browser running in development mode, refresh StockWatcher.</div>
    </li>
     <li>
        <div class="header">Add a stock code.</div>
        <div class="details">StockWatcher displays the Price and Change data. The information is now coming from a remote server.</div>
    </li>
     <li>
        <div class="header">In StockWatcher.java, change the JSON_URL so that it is not correct.</div>
    </li>
    <li>
        <div class="header">In the browser running in development mode, refresh StockWatcher.</div>
        <div class="details">Add a stock code.</div>
        <div class="details">StockWatcher displays the error: Couldn't retrieve JSON</div>
   </li>
     <li>
        <div class="header">In StockWatcher.java, correct the JSON_URL.</div>
    </li>
<li>
        <div class="header">In the browser running in development mode, refresh StockWatcher.</div>
        <div class="details">Add a stock code.</div>
        <div class="details">StockWatcher clears the error and continues displaying Price and Change updates.</div>
    </li>
</ol>
<h2>What's Next</h2>

<h3>Security and Cross-Site Requests</h3>
<p>
Before you implement mashups of your own, remember that downloading cross-site JSON is powerful, but can also be a security risk. Make sure the servers you interact with are <strong>absolutely trustworthy</strong>, because they will have the ability to execute arbitrary JavaScript code within your application. Take a minute to read <a href="../../../articles/security_for_gwt_applications.html">Security for GWT Applications</a>, which describes the potential threats to GWT applications and how you can combat them.
</p>

