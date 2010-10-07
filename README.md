The Grails remote-control plugin allows you to execute code inside a remote Grails application. The typical use case for this is for functional testing where you are testing an application inside a separate JVM and therefore do not have easy access to the application runtime. If you can access the application runtime environment then you can do things like change service parameter values, create and delete domain data and so forth.

**This plugin requires Grails 1.3.5 and will not work on earlier versions**

**There is currently no way to turn this plugin off for certain environments once installed (there will be in future versions) so DO NOT USE IN A PRODUCTION ENVIRONMENT, it is inherently a security hole**

## An Example

### The Test

We have written an application and now want to write some functional tests. In these tests we need to create some test data. This might look something like…

    class MyFunctionalTest extends GroovyTestCase {
        
        def testIt() {
            def person = new Person(name: "Me")
            person.save(flush: true)
            
            // Somehow make some HTTP request and test that person is in the DB
            
            person.delete(flush: true)
        }
        
    }

That will work if we are running our tests in the same JVM as the running application, which is the default behaviour…

    grails test-app functional:

However, it won't work if your tests ARE NOT in the same JVM, as is the case with testing against a WAR deployment…

    grails test-app functional: -war
    
This is going to fail because in the JVM that is running the tests there is no Grails application (the WAR is run in a forked JVM to be closer to a production like environment).

### Existing Solutions

The most common existing workaround for this problem is to write a special controller that you call via HTTP in your functional tests to do the setup/teardown. This will work, but requires effort and is inherently fragile.

### Using a remote control

The remote control plugin solves this problem by allowing you to define closures to be executed in the application you are testing. This is best illustrated by rewriting the above test…

    import grails.plugin.remotecontrol.RemoteControl
    
    class MyFunctionalTest extends GroovyTestCase {
        
        def remote = new RemoteControl()
        
        def testIt() {
            def id = remote {
                def person = new Person(name: "Me")
                person.save(flush: true)
                person.id
            }
            
            // Somehow make some HTTP request and test that person is in the DB
            
            remote {
                Person.get(id).delete(flush: true)
            }
        }
    }

This test will now working when testing agains a WAR or a local version. The closures passed to `remote` are sent over HTTP to the running application and executed there, so it doesn't matter where the application is.

To see some more usage examples of a remote control, see the [demonstration test case](http://github.com/alkemist/grails-remote-control/blob/master/test/functional/SmokeTests.groovy) in the project.

### Testing Remote Apps

Let's say that we want to functionally test our application on different flavours of application server and we have our app deployed on three different app servers at the following URLs:

* http://appsrv1.test.my.org/myapp
* http://appsrv2.test.my.org/myapp
* http://appsrv3.test.my.org/myapp

If we have the remote-control plugin installed and have written our tests to use it, we could simply run:

    grails test-app functional: -baseUrl=http://appsrv1.test.my.org/myapp
    grails test-app functional: -baseUrl=http://appsrv2.test.my.org/myapp
    grails test-app functional: -baseUrl=http://appsrv3.test.my.org/myapp

Which will execute the tests against that remote instance.

### Status

This plugin is currently experimental, it should not be installed in applications bound for production. However, please give it a try and let me know how you get on.