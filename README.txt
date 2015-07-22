
This issue still exists in Glassfish 4.1.

Scenario:

JAR file "ws_ejb_ok.jar" contains a Stateless session bean implementation, annotated by @Stateless, that exposes a web-service endpoint by using a @WebService annotation. 
The "name" attribute of the @Stateless annotation (value: "HelloEJBa") matches the "<ejb-name>" specified in the ejb-jar.xml descriptor.
Note that the <ejb-class> specified in the ejb-jar.xml descriptor matches the stateless session-bean implementation class.
Result: the JAR file deploys OK, the session bean and web-service endpoint are available and working. There is a message in the server.log file "EJB Endpoint deployed ws_ejb_ok listening at address at http://...", confirming that the endpoint is available.

JAR file "ws_ejb_ng.jar" is almost exactly the same as JAR file "ws_ejb_ok.jar", except that the value of the "name" attribute of the @Stateless annotation (value "HelloEJBa") DOES NOT MATCH the "<ejb-name>" specified in the ejb-jar.xml descriptor (value "HelloEJBb").
Result: the JAR file deploys OK (and no errors in the server.log), and two stateless session-beans are deployed (which I believe is correct because they have different names), BUT the web-service endpoint IS NOT AVAILABLE.
The expected message "EJB Endpoint deployed ws_ejb_ng listening at address at http://..." DOES NOT APPEAR in the server.log file.
Effectively the web-service is disabled by defining another session bean in the ejb-jar.xml descriptor that has the same class but different ejb-name.


Resolution:

I tracked down the problem to be related to the fact that when there are multiple EJBs with the same class in the one module, their processing contexts are encapsulated in an "EjbsContext" rather than a single "EjbContext".
The following Glassfish 4.1 Java class, which processes the "javax.jws.WebService" annotation, doesn't handle "EjbsContext" when it checks the type of web-service implementation (EJB/Servlet):

    4.1/appserver/webservices/connector/src/main/java/org/glassfish/webservices/connector/annotation/handlers/WebServiceHandler.java
	
The code in this class's "processAnnotation(AnnotationInfo annInfo)" method has the following form:

            if ((ejbProvider!= null) && ejbProvider.getType("javax.ejb.Stateless")!=null && (annCtx instanceof EjbContext)) {
				// this is an ejb !
				...
			else {
                // this has to be a servlet since there is no @Servlet annotation yet
				...
			}
			
So when an "EjbsContext" (as opposed to a "EjbContext") is passed as part of the "annInfo" parameter, we end up in the code block with comment "this has to be a servlet ...", which is INCORRECT.


I have produced a PATCH for this issue (for Glassfish 4.1), to be reviewed by Oracle for correctness and completeness.
Please see file: WebServiceHandler.patch

As a convenience for testing this patch, I have provided the patched JAR file "webservices-connector.jar", which is the same as the original Glassfish 4.1 JAR file but with my patched class built into it. It is a replacement for the JAR file of that name located in the "glassfish4/glassfish/modules" directory.


Summary of files:

ws_ejb_ok.jar				- EJB-based web-service that deploys and works OK
ws_ejb_ng.jar				- Same EJB-based web-service but with different <ejb-name> in ejb-jar.xml; deploys OK but no web-service endpoint is exposed
WebServiceHandler.patch		- my proposed patch (to be reviewed by Oracle)
webservices-connector.jar	- Glassfish 4.1 library JAR file which includes my patched class (org.glassfish.webservices.connector.annotation.handlers.WebServiceHandler.class)
README.txt					- this README file


