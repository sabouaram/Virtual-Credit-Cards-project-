# CedarPay Platform - Frontend


## Project Structure
The webapp should be responsive meaning it have to work correctly on all the available devices browsers.
Our project follows the Cubits -> Repositories -> Main Design Pattern for state management. It also uses the MultiblocRepositories and MultiblocProviders pattern for efficient state management.

- `lib/`
  - `cubits/`: Contains Cubit classes for state management.
  - `repositories/`: Houses repository classes for data handling.
  - `widgets/`: Reusable UI components and widgets.
  - `main.dart`
  - `config file`: JSON or YAML.

## Routing
We use the GoRouter package for URL navigation. Make sure to familiarize yourself with its usage and documentation.

## URL Error Handling
All unexpected errors should return the current URL.

## Token Management
For token management, we use `flutter_secure_storage` for mobile apps. Tokens should be securely stored on the user's device. For web applications, tokens should be managed accordingly. `To identify how to store the JWT: Mainly looking for a good and secure solution`.

We have provided a JwtTokenManager class as an example for token management. Please adapt it as needed for your specific requirements.

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class JwtTokenManager {
  final FlutterSecureStorage _secureStorage = FlutterSecureStorage();
  late String _token;

  JwtTokenManager() {
    _token = '';
    getToken().then((token) {
      if (token != null) {
        _token = token;
      }
    });
  }

  Future<String?> getToken() async {
    return await _secureStorage.read(key: 'jwt_token');
  }

  Future<void> storeToken(String token) async {
    await _secureStorage.write(key: 'jwt_token', value: token);
    _token = token;
  }

  Future<void> clearToken() async {
    await _secureStorage.delete(key: 'jwt_token');
    _token = '';
  }

  Map<String, String> getAuthMetadata() {
    if (_token.isNotEmpty) {
      return {'Authorization': 'Bearer $_token'};
    }
    return {};
  }
}
```

## gRPC API Integration
To interact with the gRPC API, we use a `BaseRepository` class that automatically handles token inclusion in the headers for authenticated requests. Extend this class for other repositories as needed.
Here's an example of how to use it:

```dart
import 'package:grpc/grpc.dart';
import './token_repository.dart';
import 'package:silvermobi/repositories/silvermobi_grpc_service.dart';
import 'package:global_configuration/global_configuration.dart';
import 'package:silvermobi/proto/silvermobi-service.pbgrpc.dart';

GlobalConfiguration cfg = GlobalConfiguration();

class BaseRepository {
  final JwtTokenManager jwtTokenManager;
  late final SilvermobiGRPCClient client;

  BaseRepository(this.jwtTokenManager) {
    client = _createClient(jwtTokenManager);
  }

  SilvermobiGRPCClient _createClient(JwtTokenManager jwtTokenManager) {
    final service = SilvermobiGRPCServiceImpl();
    return service.getClient(cfg);
  }

  Future<T> makeRequest<T>(
    Future<T> Function(SilvermobiGRPCClient client, CallOptions options)
        requestFunction,
    bool includeAuth,
  ) async {
    final options = includeAuth
        ? CallOptions(metadata: jwtTokenManager.getAuthMetadata())
        : CallOptions();
    try {
      final response = await requestFunction(client, options);
      return response;
    } catch (error) {
      // Handle error
      rethrow;
    }
  }
}
```
Example for a repository:
```dart
import 'package:silvermobi/proto/silvermobi-service.pbgrpc.dart';
import './token_repository.dart';
import 'base_repository.dart';

class AuthenticationRepository extends BaseRepository {
  AuthenticationRepository(
    JwtTokenManager jwtTokenManager,
  ) : super(
          jwtTokenManager,
        );

  Future<String> login(String username, String password) async {
    final request = AuthLoginRequest()
      ..username = username
      ..password = password;

    try {
      final response = await makeRequest(
        (client, options) => client.authLogin(request, options: options),
        false,
      );
      final token = response.token;

      if (token.isNotEmpty) {
        await jwtTokenManager.storeToken(token);
      }
      return token;
    } catch (error) {
      return '';
    }
  }
    Future<ValidationResponse> validateAccount() async {
    try {
      return await makeRequest(
        (client, options) =>
            client.getValidation(ValidationRequest(), options: options),
        true,
      );
    } catch (error) {
      rethrow;
    }
  }
}
```
## gRPC Service Configuration
The gRPC service configuration is abstracted using the default generated service interface. Make sure to configure it correctly according to your environment. we should use the `grpc_or_grpcweb` package.
Example:
```dart
import 'package:silvermobi/proto/silvermobi-service.pbgrpc.dart';
import 'package:global_configuration/global_configuration.dart';
import 'package:grpc/grpc_or_grpcweb.dart';

abstract class SilvermobiGRPCService {
  SilvermobiGRPCClient getClient(GlobalConfiguration cfg);
}

class SilvermobiGRPCServiceImpl implements SilvermobiGRPCService {
  @override
  SilvermobiGRPCClient getClient(GlobalConfiguration cfg) {
    final grpcHost = cfg.getValue('api')['grpc']['host'] as String? ?? '';
    final grpcPort = cfg.getValue('api')['grpc']['port'] as int? ?? 0;

    final channel = GrpcOrGrpcWebClientChannel.toSeparatePorts(
      host: grpcHost,
      grpcPort: grpcPort,
      grpcTransportSecure: false,
      grpcWebPort: grpcPort,
      grpcWebTransportSecure: false,
    );

    return SilvermobiGRPCClient(channel);
  }
}
```
## The API proto files:
When building the repositories that implements these methods make sure you include the JWT in the header if the method requires it.
```
| RPC Method                            | Description                                  | Requires JWT Token |
| ------------------------------------- | -------------------------------------------- | ------------------ |
| AuthLogin(AuthLoginRequest)            | Authenticate user                            | No                 |
| AuthRegister(AuthRegisterRequest)      | Register a new user                          | No                 |
| ForgetAccount(ForgetAccountRequest)    | Request to reset account                     | No                 |
| UpdatePassword(UpdatePasswordRequest)  | Update user password                         | No                 |
| getVirtualCard(VirtualCardRequest)     | Get a virtual card                           | Yes                |
| getBalance(BalanceRequest)             | Get user balance                             | Yes                |
| getPaymentHistory(PaymentHistoryRequest) | Get payment history                        | Yes                |
| UploadFile(FileUploadRequest)          | Upload a file                                | Yes                |

```

```proto
syntax = "proto3";

import "google/protobuf/timestamp.proto"; 
service CedarPayGRPC {
rpc AuthLogin(AuthLoginRequest) returns (AuthLoginResponse) {}
rpc AuthRegister(AuthRegisterRequest) returns (AuthRegisterResponse) {}
rpc ForgetAccount(ForgetAccountRequest) returns (ForgetAccountResponse) {}
rpc UpdatePassword(UpdatePasswordRequest) returns (UpdatePasswordResponse) {}
rpc getVirtualCard(VirtualCardRequest) returns (VirtualCardResponse) {}
rpc getBalance(BalanceRequest) returns (BalanceResponse) {}
rpc getPaymentHistory(PaymentHistoryRequest) returns (PaymentHistoryResponse){}
rpc UploadFile(FileUploadRequest) returns (FileUploadResponse) {}
}

message AuthLoginRequest {
  string username = 1;
  string password = 2;
}

message AuthLoginResponse {
  string token = 1;
}

message AuthRegisterRequest {
  string email = 1;
  string password = 2;
  string first_name = 3;
  string last_name = 4;
  string phone_number = 5;
}

message AuthRegisterResponse {
  string token = 1;
}

message ForgetAccountRequest {
  string username = 1;
}

message ForgetAccountResponse {
  bool response = 1;
  string access_code = 2;
}

message UpdatePasswordRequest {
  string email = 1;
  string password = 2;
}

message UpdatePasswordResponse {
  bool response = 1;
}

message VirtualCardRequest {
  int64 amount = 1;
  int64 month = 2;
  int64 year = 3;
}

message VirtualCardResponse {
  int64 cardnumber = 1;
  int64 cvc = 2;
}

message BalanceRequest {}

message BalanceResponse {
  float balance = 1;
}

message PaymentHistoryRequest {}

message PaymentOperation {
  string merchantName = 1;
  float amount = 2;
  google.protobuf.Timestamp date = 3;
}

message PaymentHistoryResponse {
  repeated PaymentOperation operations = 1;
}

message FileUploadRequest {
  string filename = 1;
  string type = 2;
  bytes file_content = 3;
}

message FileUploadResponse {
  bool result = 1;
}

```
