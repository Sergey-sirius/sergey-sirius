---
layout: page
title: Приложение A. Коды ошибок PostgreSQL
description: ""
tags: [PostgreSQL]
image:
  feature: abstract-11.jpg
  #credit: dargadgetz
  #creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
share: true
modified: 2018-12-03 T15:14:43-04:00
---

Часть VIII. Приложения
Приложение A. Коды ошибок PostgreSQL


Всем сообщениям, которые выдаёт сервер PostgreSQL, назначены пятисимвольные коды ошибок,
соответствующие кодам «SQLSTATE», описанным в стандарте SQL. Приложения, которые должны
знать, какое условие ошибки имело место, обычно проверяют код ошибки и только потом обраща-
ются к текстовому сообщению об ошибке. Коды ошибок, скорее всего, не изменятся от выпуска к
выпуску PostgreSQL, и они не меняются при локализации как сообщения об ошибках. Заметьте,
что отдельные, но не все коды ошибок, которые выдаёт PostgreSQL, определены стандартом SQL;
некоторые дополнительные коды ошибок для условий, не описанных стандартом, были добавлены
независимо или позаимствованы из других баз данных.
Согласно стандарту, первые два символа кода ошибки обозначают класс ошибок, а последние три
символа обозначают определённое условие в этом классе. Таким образом, приложение, не знаю-
щее значение определённого кода ошибки, всё же может понять, что делать, по классу ошибки.
В Таблице A.1 перечислены все коды ошибок, определённые в PostgreSQL 11.1. (Некоторые коды в
настоящее время не используются, хотя они определены в стандарте SQL.) Также показаны клас-
сы ошибок. Для каждого класса ошибок имеется «стандартный» код ошибки с последними тремя
символами 000. Этот код выдаётся только для таких условий ошибок, которые относятся к некото-
рому классу, но не имеют более определённого кода.
Символ, указанный в столбце «Имя условия», определяет условие в PL/pgSQL. Имена условий мо-
гут записываться в верхнем или нижнем регистре. (Заметьте, что PL/pgSQL, в отличие от ошибок,
не распознаёт предупреждения; то есть классы 00, 01 и 02.)
Для некоторых типов ошибок сервер сообщает имя объекта базы данных (таблица, столбец таб-
лицы, тип данных или ограничение), связанного с ошибкой; например, имя уникального ограни-
чения, вызвавшего ошибку unique_violation. Такие имена передаются в отдельных полях сооб-
щения об ошибке, чтобы приложениям не пришлось извлекать его из возможно локализованного
текста ошибки для человека. На момент выхода PostgreSQL 9.3 полностью охватывались только
ошибки класса SQLSTATE 23 (нарушения ограничений целостности), но в будущем должны быть
охвачены и другие классы.
Таблица A.1. Коды ошибок PostgreSQL
Код ошибки
Имя условия
Класс 00 — Успешное завершение
00000
successful_completion
Класс 01 — Предупреждение
01000 warning
0100C dynamic_result_sets_returned
01008 implicit_zero_bit_padding
01003 null_value_eliminated_in_set_function
01007 privilege_not_granted
01006 privilege_not_revoked
01004 string_data_right_truncation
01P01 deprecated_feature
Класс 02 — Нет данных (это также класс предупреждений согласно стандарту SQL)
02000 no_data
02001 no_additional_dynamic_result_sets_
returned
2198Коды ошибок PostgreSQL
Код ошибки
Имя условия
Класс 03 — SQL-оператор ещё не завершён
03000
sql_statement_not_yet_complete
Класс 08 — Исключение, связанное с подключением
08000 connection_exception
08003 connection_does_not_exist
08006 connection_failure
08001 sqlclient_unable_to_establish_
sqlconnection
08004 sqlserver_rejected_establishment_of_
sqlconnection
08007 transaction_resolution_unknown
08P01 protocol_violation
Класс 09 — Исключение с действием триггера
09000
triggered_action_exception
Класс 0A — Неподдерживаемая функциональность
0A000
feature_not_supported
Класс 0B — Неверное начало транзакции
0B000
invalid_transaction_initiation
Класс 0F — Исключение с указателем на данные
0F000 locator_exception
0F001 invalid_locator_specification
Класс 0L — Неверный праводатель
0L000 invalid_grantor
0LP01 invalid_grant_operation
Класс 0P — Неверное указание роли
0P000
invalid_role_specification
Класс 0Z — Исключение диагностики
0Z000 diagnostics_exception
0Z002 stacked_diagnostics_accessed_without_
active_handler
Класс 20 — Case не найден
20000
case_not_found
Класс 21 — Нарушение количества
21000
cardinality_violation
Класс 22 — Исключение в данных
22000 data_exception
2202E array_subscript_error
22021 character_not_in_repertoire
22008 datetime_field_overflow
22012 division_by_zero
22005 error_in_assignment
2199Коды ошибок PostgreSQL
Код ошибки Имя условия
2200B escape_character_conflict
22022 indicator_overflow
22015 interval_field_overflow
2201E invalid_argument_for_logarithm
22014 invalid_argument_for_ntile_function
22016 invalid_argument_for_nth_value_
function
2201F invalid_argument_for_power_function
2201G invalid_argument_for_width_bucket_
function
22018 invalid_character_value_for_cast
22007 invalid_datetime_format
22019 invalid_escape_character
2200D invalid_escape_octet
22025 invalid_escape_sequence
22P06 nonstandard_use_of_escape_character
22010 invalid_indicator_parameter_value
22023 invalid_parameter_value
22013 invalid_preceding_or_following_size
2201B invalid_regular_expression
2201W invalid_row_count_in_limit_clause
2201X invalid_row_count_in_result_offset_
clause
2202H invalid_tablesample_argument
2202G invalid_tablesample_repeat
22009 invalid_time_zone_displacement_value
2200C invalid_use_of_escape_character
2200G most_specific_type_mismatch
22004 null_value_not_allowed
22002 null_value_no_indicator_parameter
22003 numeric_value_out_of_range
2200H sequence_generator_limit_exceeded
22026 string_data_length_mismatch
22001 string_data_right_truncation
22011 substring_error
22027 trim_error
22024 unterminated_c_string
2200F zero_length_character_string
22P01 floating_point_exception
22P02 invalid_text_representation
22P03 invalid_binary_representation
2200Коды ошибок PostgreSQL
Код ошибки Имя условия
22P04 bad_copy_file_format
22P05 untranslatable_character
2200L not_an_xml_document
2200M invalid_xml_document
2200N invalid_xml_content
2200S invalid_xml_comment
2200T invalid_xml_processing_instruction
Класс 23 — Нарушение ограничения целостности
23000 integrity_constraint_violation
23001 restrict_violation
23502 not_null_violation
23503 foreign_key_violation
23505 unique_violation
23514 check_violation
23P01 exclusion_violation
Класс 24 — Неверное состояние курсора
24000
invalid_cursor_state
Класс 25 — Неверное состояние транзакции
25000 invalid_transaction_state
25001 active_sql_transaction
25002 branch_transaction_already_active
25008 held_cursor_requires_same_isolation_
level
25003 inappropriate_access_mode_for_branch_
transaction
25004 inappropriate_isolation_level_for_
branch_transaction
25005 no_active_sql_transaction_for_branch_
transaction
25006 read_only_sql_transaction
25007 schema_and_data_statement_mixing_not_
supported
25P01 no_active_sql_transaction
25P02 in_failed_sql_transaction
25P03 idle_in_transaction_session_timeout
Класс 26 — Неверное имя SQL-оператора
26000
invalid_sql_statement_name
Класс 27 — Нарушение при изменении данных в триггере
27000
triggered_data_change_violation
Класс 28 — Неверное указание авторизации
28000
invalid_authorization_specification
2201Коды ошибок PostgreSQL
Код ошибки Имя условия
28P01 invalid_password
Класс 2B — Зависимые описания привилегий всё ещё существуют
2B000 dependent_privilege_descriptors_still_
exist
2BP01 dependent_objects_still_exist
Класс 2D — Неверное завершение транзакции
2D000
invalid_transaction_termination
Класс 2F — Исключение в подпрограмме SQL
2F000 sql_routine_exception
2F005 function_executed_no_return_statement
2F002 modifying_sql_data_not_permitted
2F003 prohibited_sql_statement_attempted
2F004 reading_sql_data_not_permitted
Класс 34 — Неверное имя курсора
34000
invalid_cursor_name
Класс 38 — Исключение во внешней подпрограмме
38000 external_routine_exception
38001 containing_sql_not_permitted
38002 modifying_sql_data_not_permitted
38003 prohibited_sql_statement_attempted
38004 reading_sql_data_not_permitted
Класс 39 — Исключение при вызове внешней подпрограммы
39000 external_routine_invocation_exception
39001 invalid_sqlstate_returned
39004 null_value_not_allowed
39P01 trigger_protocol_violated
39P02 srf_protocol_violated
39P03 event_trigger_protocol_violated
Класс 3B — Исключение точки сохранения
3B000 savepoint_exception
3B001 invalid_savepoint_specification
Класс 3D — Неверное имя каталога
3D000
invalid_catalog_name
Класс 3F — Неверное имя схемы
3F000
invalid_schema_name
Класс 40 — Откат транзакции
40000 transaction_rollback
40002 transaction_integrity_constraint_
violation
40001 serialization_failure
40003 statement_completion_unknown
2202Коды ошибок PostgreSQL
Код ошибки Имя условия
40P01 deadlock_detected
Класс 42 — Ошибка синтаксиса или нарушение правила доступа
42000 syntax_error_or_access_rule_violation
42601 syntax_error
42501 insufficient_privilege
42846 cannot_coerce
42803 grouping_error
42P20 windowing_error
42P19 invalid_recursion
42830 invalid_foreign_key
42602 invalid_name
42622 name_too_long
42939 reserved_name
42804 datatype_mismatch
42P18 indeterminate_datatype
42P21 collation_mismatch
42P22 indeterminate_collation
42809 wrong_object_type
428C9 generated_always
42703 undefined_column
42883 undefined_function
42P01 undefined_table
42P02 undefined_parameter
42704 undefined_object
42701 duplicate_column
42P03 duplicate_cursor
42P04 duplicate_database
42723 duplicate_function
42P05 duplicate_prepared_statement
42P06 duplicate_schema
42P07 duplicate_table
42712 duplicate_alias
42710 duplicate_object
42702 ambiguous_column
42725 ambiguous_function
42P08 ambiguous_parameter
42P09 ambiguous_alias
42P10 invalid_column_reference
42611 invalid_column_definition
42P11 invalid_cursor_definition
2203Коды ошибок PostgreSQL
Код ошибки Имя условия
42P12 invalid_database_definition
42P13 invalid_function_definition
42P14 invalid_prepared_statement_definition
42P15 invalid_schema_definition
42P16 invalid_table_definition
42P17 invalid_object_definition
Класс 44 — Нарушение WITH CHECK OPTION
44000
with_check_option_violation
Класс 53 — Нехватка ресурсов
53000 insufficient_resources
53100 disk_full
53200 out_of_memory
53300 too_many_connections
53400 configuration_limit_exceeded
Класс 54 — Превышение ограничения программы
54000 program_limit_exceeded
54001 statement_too_complex
54011 too_many_columns
54023 too_many_arguments
Класс 55 — Объект не в требуемом состоянии
55000 object_not_in_prerequisite_state
55006 object_in_use
55P02 cant_change_runtime_param
55P03 lock_not_available
Класс 57 — Вмешательство оператора
57000 operator_intervention
57014 query_canceled
57P01 admin_shutdown
57P02 crash_shutdown
57P03 cannot_connect_now
57P04 database_dropped
Класс 58 — Ошибка системы (ошибка, внешняя по отношению к PostgreSQL)
58000 system_error
58030 io_error
58P01 undefined_file
58P02 duplicate_file
Класс 72 — Ошибка снимка
72000
snapshot_too_old
Класс F0 — Ошибка файла конфигурации
F0000
config_file_error
2204Коды ошибок PostgreSQL
Код ошибки Имя условия
F0001 lock_file_exists
Класс HV — Ошибка обёртки сторонних данных (SQL/MED)
HV000 fdw_error
HV005 fdw_column_name_not_found
HV002 fdw_dynamic_parameter_value_needed
HV010 fdw_function_sequence_error
HV021 fdw_inconsistent_descriptor_information
HV024 fdw_invalid_attribute_value
HV007 fdw_invalid_column_name
HV008 fdw_invalid_column_number
HV004 fdw_invalid_data_type
HV006 fdw_invalid_data_type_descriptors
HV091 fdw_invalid_descriptor_field_
identifier
HV00B fdw_invalid_handle
HV00C fdw_invalid_option_index
HV00D fdw_invalid_option_name
HV090 fdw_invalid_string_length_or_buffer_
length
HV00A fdw_invalid_string_format
HV009 fdw_invalid_use_of_null_pointer
HV014 fdw_too_many_handles
HV001 fdw_out_of_memory
HV00P fdw_no_schemas
HV00J fdw_option_name_not_found
HV00K fdw_reply_handle
HV00Q fdw_schema_not_found
HV00R fdw_table_not_found
HV00L fdw_unable_to_create_execution
HV00M fdw_unable_to_create_reply
HV00N fdw_unable_to_establish_connection
Класс P0 — Ошибка PL/pgSQL
P0000 plpgsql_error
P0001 raise_exception
P0002 no_data_found
P0003 too_many_rows
P0004 assert_failure
Класс XX — Внутренняя ошибка
XX000 internal_error
XX001 data_corrupted
2205Коды ошибок PostgreSQL
Код ошибки Имя условия
XX002 index_corrupted
2206
