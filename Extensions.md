<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at
      http://www.apache.org/licenses/LICENSE-2.0
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
# Apache MiNiFi Extensions HowTo

Extensions consist of modules that are conditionally built into your client. Reasons why you may wish to do this with your modules/processors

  - Do not with to make dependencies required or the lack thereof is a known/expected runtime condition.
  - You wish to allow users to exclude dependencies for a variety of reasons.

# Extensions by example
We've used HTTP-CURL as the first example. We've taken all libcURL runtime classes and placed them into an extensions folder 
   - /extensions/http-curl
   
This folder contains a CMakeLists file so that we can conditionally build it. In the case with libcURL, if the user does not have curl installed OR they specify -DDISABLE_CURL=true in the cmake build, the extensions will not be built. In this case, when the extension is not built, C2 REST protocols, InvokeHTTP, and an HTTP Client implementation will not be included.

Your CMAKE file should build a static library, that will be included into the run time. This must be added with your conditional to the libminifi CMAKE, along with a platform specific whole archive inclusion. Note that this will ensure that despite no direct linkage being found by the compiler, we will include the code so that we can dynamically find your code.

# C bindings
To find your classes, you must adhere to a dlsym call back that adheres to the core::ObjectFactory class, like the one below. This object factory will return a list of classes, that we can instantiate through the class loader mechanism. Note that since we are including your code directly into our runtime, we will take care of dlopen and dlsym calls. A map from the class name to the object factory is kept in memory.

```C++
class __attribute__((visibility("default"))) HttpCurlObjectFactory : public core::ObjectFactory {
 public:
  HttpCurlObjectFactory() {

  }

  /**
   * Gets the name of the object.
   * @return class name of processor
   */
  virtual std::string getName() {
    return "HttpCurlObjectFactory";
  }

  virtual std::string getClassName() {
    return "HttpCurlObjectFactory";
  }
  /**
   * Gets the class name for the object
   * @return class name for the processor.
   */
  virtual std::vector<std::string> getClassNames() {
    std::vector<std::string> class_names;
    class_names.push_back("RESTProtocol");
    class_names.push_back("RESTReceiver");
    class_names.push_back("RESTSender");
    class_names.push_back("InvokeHTTP");
    class_names.push_back("HTTPClient");
    return class_names;
  }

  virtual std::unique_ptr<ObjectFactory> assign(const std::string &class_name) {
    if (class_name == "RESTReceiver") {
      return std::unique_ptr<ObjectFactory>(new core::DefautObjectFactory<minifi::c2::RESTReceiver>());
    } else if (class_name == "RESTSender") {
      return std::unique_ptr<ObjectFactory>(new core::DefautObjectFactory<minifi::c2::RESTSender>());
    } else if (class_name == "InvokeHTTP") {
      return std::unique_ptr<ObjectFactory>(new core::DefautObjectFactory<processors::InvokeHTTP>());
    } else if (class_name == "HTTPClient") {
      return std::unique_ptr<ObjectFactory>(new core::DefautObjectFactory<utils::HTTPClient>());
    } else {
      return nullptr;
    }
  }

};

extern "C" {
void *createHttpCurlFactory(void);
}
```

#Using your object factory function
To use your C function, you must define the sequence "Class Loader Functions" in your YAML file under FlowController. This will indicate to the code that the factory function is to be called and we will create the mappings defined, above.

Note that for the case of HTTP-CURL we have made it so that if libcURL exists and it is not disabled, the createHttpCurlFactory function is automatically loaded. To do this use the function FlowConfiguration::add_static_func -- this will add your function to the list of registered resources and will do so in a thread safe way. If you take this approach you cannot disable your library with an argument within the YAML file.

