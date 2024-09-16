---

# **API Service in Flutter: A Complete Documentation**

## **Introduction**

In modern mobile applications, interaction with RESTful APIs is a common task. Handling network requests efficiently and gracefully is essential for building a robust, user-friendly Flutter app. This documentation describes how to handle REST APIs in Flutter using industry best practices, ensuring proper error handling, response management, retries, logging, and testing.

## **Table of Contents**

1. [Setting Up the API Service](#setup)
2. [Error Handling](#error_handling)
   - HTTP Status Codes
   - Network Errors
3. [Timeouts and Retry Mechanism](#timeouts_retries)
4. [Parsing and Validating Responses](#parsing)
5. [Custom Exceptions](#custom_exceptions)
6. [Dio and Interceptors](#interceptors)
7. [Caching and Offline Support](#caching)
8. [Logging](#logging)
9. [Testing](#testing)
10. [Example Code](#example)

---

## <a name="setup">1. Setting Up the API Service</a>

The first step in managing API requests in Flutter is to create a **dedicated API service** class. This ensures that all your API-related logic is encapsulated in a single place, making it easier to manage and test.

### **Creating the API Service Class**

```dart
import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;

class ApiService {
  final String baseUrl = "https://api.example.com";
  final http.Client client;

  ApiService({required this.client});

  Future<dynamic> get(String endpoint) async {
    try {
      final response = await client.get(Uri.parse("$baseUrl/$endpoint"))
          .timeout(Duration(seconds: 10));
      return _handleResponse(response);
    } on SocketException {
      throw NoInternetException();
    } on TimeoutException {
      throw TimeoutException();
    } catch (e) {
      throw UnknownException();
    }
  }

  // Add POST, PUT, DELETE requests here in a similar way.
}
```

### **Separation of Concerns**

- **Service Layer**: Handles all network requests (GET, POST, PUT, DELETE).
- **UI Layer**: Displays data and handles state; calls the service layer asynchronously.

This structure allows you to keep your API logic modular, maintainable, and easy to test.

---

## <a name="error_handling">2. Error Handling</a>

Handling errors gracefully ensures that users are informed when something goes wrong, such as no internet connection or server issues.

### **Handling HTTP Status Codes**

In a real-world scenario, different HTTP status codes represent various situations. We handle these using condition checks based on the status code.

```dart
dynamic _handleResponse(http.Response response) {
  if (response.statusCode >= 200 && response.statusCode < 300) {
    // Success: Return the decoded response body
    return jsonDecode(response.body);
  } else if (response.statusCode == 400) {
    throw BadRequestException(response.body);
  } else if (response.statusCode == 401) {
    throw UnauthorizedException();
  } else if (response.statusCode == 500) {
    throw ServerException();
  } else {
    throw HttpException('Unexpected error: ${response.statusCode}');
  }
}
```

### **Network and Timeout Errors**

Use try-catch blocks to handle errors like network failures (no internet connection) or request timeouts.

```dart
void _handleError(dynamic e) {
  if (e is SocketException) {
    throw NoInternetException();
  } else if (e is TimeoutException) {
    throw TimeoutException();
  } else {
    throw UnknownException();
  }
}
```

These checks ensure the app responds appropriately depending on the error type.

---

## <a name="timeouts_retries">3. Timeouts and Retry Mechanism</a>

Timeouts and retries ensure that your app doesn't wait indefinitely for a response and can automatically retry failed requests when necessary.

### **Timeout Handling**

By default, requests can hang indefinitely if there is no response. Add timeouts to prevent this.

```dart
Future<dynamic> get(String endpoint) async {
  try {
    final response = await client.get(Uri.parse("$baseUrl/$endpoint"))
        .timeout(Duration(seconds: 10));
    return _handleResponse(response);
  } catch (e) {
    _handleError(e);
    rethrow;
  }
}
```

### **Retry Mechanism**

Implement a retry mechanism to handle temporary network issues or server downtimes.

```dart
Future<http.Response> getWithRetry(String endpoint) async {
  final retryOptions = RetryOptions(maxAttempts: 3);
  return retryOptions.retry(
    () => get(endpoint),
    retryIf: (e) => e is SocketException || e is TimeoutException,
  );
}
```

Using retry libraries like `retry` helps simplify retry logic.

---

## <a name="parsing">4. Parsing and Validating Responses</a>

Ensure proper parsing and validation of responses to avoid runtime crashes or data inconsistencies.

### **Parsing JSON Response**

```dart
dynamic parseResponse(http.Response response) {
  final json = jsonDecode(response.body);
  if (json is Map<String, dynamic>) {
    // Handle response
    return json;
  } else {
    throw InvalidFormatException();
  }
}
```

### **Error Handling for Invalid Responses**

Throw custom exceptions when the response is not in the expected format.

```dart
class InvalidFormatException implements Exception {
  final String message;
  InvalidFormatException([this.message = "Invalid response format"]);
}
```

---

## <a name="custom_exceptions">5. Custom Exceptions</a>

Use custom exceptions to handle various error scenarios, making it easier to debug and manage errors in your application.

### **Example Custom Exceptions**

```dart
class NoInternetException implements Exception {
  final String message;
  NoInternetException([this.message = "No internet connection"]);
}

class TimeoutException implements Exception {
  final String message;
  TimeoutException([this.message = "Request timed out"]);
}

class UnauthorizedException implements Exception {}
class ServerException implements Exception {}
```

By throwing custom exceptions, you can handle specific cases gracefully, such as redirecting users to a login screen if they encounter an unauthorized error.

---

## <a name="interceptors">6. Dio and Interceptors</a>

Using **Dio** provides advanced functionality such as interceptors, which are ideal for adding common request logic, like logging or retrying.

### **Example with Dio Interceptors**

```dart
Dio dio = Dio();

dio.interceptors.add(InterceptorsWrapper(
  onRequest: (options, handler) {
    // Add headers, logging, etc.
    return handler.next(options);
  },
  onResponse: (response, handler) {
    // Process response globally
    return handler.next(response);
  },
  onError: (DioError error, handler) {
    // Handle errors globally
    return handler.next(error);
  },
));
```

Interceptors allow you to handle request and response logic globally, such as refreshing tokens or logging every request.

---

## <a name="caching">7. Caching and Offline Support</a>

Caching is essential for optimizing performance, especially when users have limited connectivity.

### **Dio Cache Example**

Use Dio's caching interceptors or libraries like `flutter_cache_manager` to cache responses.

```dart
dio.interceptors.add(DioCacheManager(CacheConfig()).interceptor);
```

This allows the app to fetch cached data when the device is offline.

---

## <a name="logging">8. Logging</a>

Logging API requests and responses helps with debugging and monitoring the app's network behavior.

### **Using Dio's Log Interceptor**

```dart
dio.interceptors.add(LogInterceptor(responseBody: true));
```

You can also use the **logger** package for additional logging capabilities.

---

## <a name="testing">9. Testing</a>

Testing your API logic ensures that error handling and business logic work as expected.

### **Mocking API Requests**

Use libraries like `http_mock_adapter` (for Dio) or `mockito` (for http) to mock API responses during testing.

```dart
@GenerateMocks([http.Client])
void main() {
  group('ApiService Test', () {
    test('returns data if the http call completes successfully', () async {
      final client = MockClient();
      when(client.get(any)).thenAnswer(
        (_) async => http.Response('{"data": "example"}', 200),
      );

      final apiService = ApiService(client: client);
      final result = await apiService.get('/example');
      expect(result, {'data': 'example'});
    });
  });
}
```

This approach helps simulate different scenarios (success, failure, etc.) during unit testing.

---

## <a name="example">10. Example Code</a>

### **Complete ApiService Class**

```dart
import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;

class ApiService {
  final String baseUrl = "https://api.example.com";
  final http.Client client;

  ApiService({required this.client});

  Future<dynamic> get(String endpoint) async {
    try {
      final response = await client.get(Uri.parse("$baseUrl/$endpoint"))
          .timeout(Duration(seconds: 10));
      return _handleResponse(response);
    } on SocketException {
      throw NoInternetException();
    } on TimeoutException {
      throw TimeoutException();
    } catch (e) {
      throw UnknownException();
    }
  }

  dynamic _handleResponse(http.Response response) {
    if (response.statusCode >= 

200 && response.statusCode < 300) {
      return jsonDecode(response.body);
    } else if (response.statusCode == 400) {
      throw BadRequestException(response.body);
    } else if (response.statusCode == 401) {
      throw UnauthorizedException();
    } else if (response.statusCode == 500) {
      throw ServerException();
    } else {
      throw HttpException('Unexpected error: ${response.statusCode}');
    }
  }
}

class NoInternetException implements Exception {
  final String message;
  NoInternetException([this.message = "No internet connection"]);
}

class TimeoutException implements Exception {
  final String message;
  TimeoutException([this.message = "Request timed out"]);
}

class UnauthorizedException implements Exception {}
class ServerException implements Exception {}
```

---

## **Conclusion**

This documentation outlines best practices for handling REST APIs in Flutter, from managing errors and timeouts to using custom exceptions, interceptors, and caching mechanisms. By following this structure, your Flutter app will handle API requests in a reliable, maintainable, and scalable manner.
