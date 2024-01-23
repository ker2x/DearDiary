# Fastfail Codes

https://www.softwareverify.com/blog/fail-fast-codes/

| Definition	                                | Value | 	Comment                                  |
|--------------------------------------------|-------|-------------------------------------------|
| FAST_FAIL_LEGACY_GS_VIOLATION	             | 0     | 	Do not use. Legacy value.                |
| FAST_FAIL_VTGUARD_CHECK_FAILURE	           | 1     |                                           |
| FAST_FAIL_STACK_COOKIE_CHECK_FAILURE	      | 2     |                                           |
| FAST_FAIL_CORRUPT_LIST_ENTRY	              | 3     |                                           |
| FAST_FAIL_INCORRECT_STACK	                 | 4     |                                           |
| FAST_FAIL_INVALID_ARG	                     | 5     |                                           |
| FAST_FAIL_GS_COOKIE_INIT	                  | 6     |                                           |
| FAST_FAIL_FATAL_APP_EXIT	                  | 7     |                                           |
| FAST_FAIL_RANGE_CHECK_FAILURE	             | 8     |                                           |
| FAST_FAIL_UNSAFE_REGISTRY_ACCESS	          | 9     |                                           |
| FAST_FAIL_GUARD_ICALL_CHECK_FAILURE	       | 10    |                                           |
| FAST_FAIL_GUARD_WRITE_CHECK_FAILURE	       | 11    |                                           |
| FAST_FAIL_INVALID_FIBER_SWITCH	            | 12    |                                           |
| FAST_FAIL_INVALID_SET_OF_CONTEXT	          | 13    |                                           |
| FAST_FAIL_INVALID_REFERENCE_COUNT	         | 14    |                                           |
| FAST_FAIL_INVALID_JUMP_BUFFER	             | 18    |                                           |
| FAST_FAIL_MRDATA_MODIFIED	                 | 19    |                                           |
| FAST_FAIL_CERTIFICATION_FAILURE	           | 20    |                                           |
| FAST_FAIL_INVALID_EXCEPTION_CHAIN	         | 21    |                                           |
| FAST_FAIL_CRYPTO_LIBRARY	                  | 22    |                                           |
| FAST_FAIL_INVALID_CALL_IN_DLL_CALLOUT	     | 23    |                                           |
| FAST_FAIL_INVALID_IMAGE_BASE	              | 24    |                                           |
| FAST_FAIL_DLOAD_PROTECTION_FAILURE	        | 25    |                                           |
| FAST_FAIL_UNSAFE_EXTENSION_CALL	           | 26    |                                           |
| FAST_FAIL_DEPRECATED_SERVICE_INVOKED	      | 27    |                                           |
| FAST_FAIL_INVALID_BUFFER_ACCESS	           | 28    |                                           |
| FAST_FAIL_INVALID_BALANCED_TREE	           | 29    |                                           |
| FAST_FAIL_INVALID_NEXT_THREAD	             | 30    |                                           |
| FAST_FAIL_GUARD_ICALL_CHECK_SUPPRESSED	    | 31    | 	Telemetry, nonfatal                      |
| FAST_FAIL_APCS_DISABLED	                   | 32    |                                           |
| FAST_FAIL_INVALID_IDLE_STATE	              | 33    |                                           |
| FAST_FAIL_MRDATA_PROTECTION_FAILURE	       | 34    |                                           |
| FAST_FAIL_UNEXPECTED_HEAP_EXCEPTION	       | 35    |                                           |
| FAST_FAIL_INVALID_LOCK_STATE	              | 36    |                                           |
| FAST_FAIL_GUARD_JUMPTABLE	                 | 37    | 	Compiler uses this value. Do not change. |
| FAST_FAIL_INVALID_LONGJUMP_TARGET	         | 38    |                                           |
| FAST_FAIL_INVALID_DISPATCH_CONTEXT	        | 39    |                                           |
| FAST_FAIL_INVALID_THREAD	                  | 40    |                                           |
| FAST_FAIL_INVALID_SYSCALL_NUMBER	          | 41    | 	Telemetry, nonfatal                      |
| FAST_FAIL_INVALID_FILE_OPERATION	          | 42    | 	Telemetry, nonfatal                      |
| FAST_FAIL_LPAC_ACCESS_DENIED	              | 43    | 	Telemetry, nonfatal                      |
| FAST_FAIL_GUARD_SS_FAILURE	                | 44    |                                           |
| FAST_FAIL_LOADER_CONTINUITY_FAILURE	       | 45    | 	Telemetry, nonfatal                      |
| FAST_FAIL_GUARD_EXPORT_SUPPRESSION_FAILURE | 46    |                                           |
| FAST_FAIL_INVALID_CONTROL_STACK	           | 47    |                                           |
| FAST_FAIL_SET_CONTEXT_DENIED	              | 48    |                                           |
| FAST_FAIL_INVALID_IAT	                     | 49    |                                           |
| FAST_FAIL_HEAP_METADATA_CORRUPTION	        | 50    |                                           |
| FAST_FAIL_PAYLOAD_RESTRICTION_VIOLATION	   | 51    |                                           |
| FAST_FAIL_LOW_LABEL_ACCESS_DENIED	         | 52    | 	Telemetry, nonfatal                      |
| FAST_FAIL_ENCLAVE_CALL_FAILURE	            | 53    |                                           |
| FAST_FAIL_UNHANDLED_LSS_EXCEPTON	          | 54    |                                           |
| FAST_FAIL_ADMINLESS_ACCESS_DENIED	         | 55    | 	Telemetry, nonfatal                      |
| FAST_FAIL_UNEXPECTED_CALL	                 | 56    |                                           |
| FAST_FAIL_CONTROL_INVALID_RETURN_ADDRESS	  | 57    |                                           |
| FAST_FAIL_UNEXPECTED_HOST_BEHAVIOR	        | 58    |                                           |
| FAST_FAIL_FLAGS_CORRUPTION	                | 59    |                                           |
| FAST_FAIL_VEH_CORRUPTION	                  | 60    |                                           |
| FAST_FAIL_ETW_CORRUPTION	                  | 61    |                                           |
| FAST_FAIL_RIO_ABORT	                       | 62    |                                           |
| FAST_FAIL_INVALID_PFN	                     | 63    |                                           |
| FAST_FAIL_GUARD_ICALL_CHECK_FAILURE_XFG	   | 64    |                                           |
| FAST_FAIL_CAST_GUARD	                      | 65    | 	Compiler uses this value. Do not change. |
| FAST_FAIL_HOST_VISIBILITY_CHANGE	          | 66    |                                           |
| FAST_FAIL_KERNEL_CET_SHADOW_STACK_ASSIST	  | 67    |                                           |
| FAST_FAIL_PATCH_CALLBACK_FAILED	           | 68    |                                           |
| FAST_FAIL_NTDLL_PATCH_FAILED	              | 69    |                                           |
| FAST_FAIL_INVALID_FLS_DATA	                | 70    |                                           |