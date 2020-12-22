# List of possible errors

| gRPC code | gRPC status         | HTTP code | Error description                                            |
| :-------- | :------------------ | :-------- | :----------------------------------------------------------- |
| 1         | CANCELLED           | 499       | The operation was aborted on the client side.                |
| 2         | UNKNOWN             | 500       | Unknown error.                                               |
| 3         | INVALID_ARGUMENT    | 400       | Incorrect request parameters specified. Details are provided in the `details` field. |
| 4         | DEADLINE_EXCEEDED   | 504       | Exceeded the server response timeout.                        |
| 5         | NOT_FOUND           | 404       | The requested resource not found.                            |
| 6         | ALREADY_EXISTS      | 409       | Attempt to create a resource that already exists.            |
| 7         | PERMISSION_DENIED   | 403       | The user has no permissions required to perform the operation. |
| 8         | RESOURCE_EXHAUSTED  | 429       | The request limit exceeded.                                  |
| 9         | FAILED_PRECONDITION | 400       | The operation was canceled because the conditions required for the operation were not met. Examples: attempting to delete a non-empty folder or calling the rmdir command for an object that is not a folder. |
| 10        | ABORTED             | 409       | The operation failed due to a concurrent computing conflict, such as an invalid sequence of commands or an aborted transaction. |
| 11        | OUT_OF_RANGE        | 400       | Out of range. For example, searching or reading outside of the file. |
| 12        | NOT_IMPLEMENTED     | 501       | The operation is not supported by the service.               |
| 13        | INTERNAL            | 500       | Internal server error. This error means that the operation cannot be performed due to a server-side technical problem. For example, due to insufficient computing resources. |
| 14        | UNAVAILABLE         | 503       | The service is currently unavailable. Try again in a few seconds. |
| 15        | DATA_LOSS           | 500       | Permanent data loss or damage.                               |
| 16        | UNAUTHENTICATED     | 401       | The operation requires authentication.                       |



参考资料

https://googleapis.dev/python/google-api-core/latest/exceptions.html

https://cloud.yandex.com/docs/api-design-guide/concepts/errors