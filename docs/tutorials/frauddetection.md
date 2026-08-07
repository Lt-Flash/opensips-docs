---
title: "Using the Fraud Detection module"
subtitle: "Fraud Detection"
subtitleHref: "/manual/devel/modules/fraud_detection"
description: "Fraudulent calls have been a part of VoIP since its very beginnings. Typically, there are two ways through which a malicious user can gain permission to plac..."
---

## Introduction

Fraudulent calls have been a part of VoIP since its very beginnings.
Typically, there are two ways through which a malicious user can gain
permission to place calls through a VoIP platform: *account hijacking* and
*weak platform security*.

The module provides a way of detecting suspicious calls from the same users to
the same destinations that would otherwise likely end up costing the affected
VoIP account, or the VoIP provider, considerable amounts of money. This is done
by closely monitoring a set of call-related parameters for each
**[user,destination]** pair, together with provisioned alerting thresholds.

## Module Overview

### Functionality

With the help of the dialog module, **fraud_detection** can monitor the
following call-related statistics for each *[user,destination]* pair:

* calls per minute
* call duration
* total calls
* concurrent calls
* consecutive calls to the same destination

The statistics are evaluated against user-provisioned time intervals and
thresholds. See [Table provisioning](#table-provisioning) below.

### DB provisioning

#### Table creation

The **fraud_detection** table is not part of the standard set of tables created
by **opensips-cli**, unless you provide the `database_modules: ALL` setting in
your `opensips-cli.cfg` file during database creation. If you do not already
have the table, add it as follows:

```bash
opensips-cli -x database add fraud_detection
```

The module also depends on **drouting** and **dialog**. If this is a fresh
database and those tables are not already present, add their schemas as well, or
create the database with `database_modules: ALL`.

#### Table provisioning

The following is an example of a fraud detection profile for destination prefix
**99**. All rules below are part of the same profile: **1**.

```sql
INSERT INTO fraud_detection (
    profileid, prefix, start_hour, end_hour, daysoftheweek,
    cpm_warning, cpm_critical,
    call_duration_warning, call_duration_critical,
    total_calls_warning, total_calls_critical,
    concurrent_calls_warning, concurrent_calls_critical,
    sequential_calls_warning, sequential_calls_critical
) VALUES
    (1, '99', '09:00', '17:00', 'Mon-Fri', 3, 5, 7200, 13200, 16, 35, 3, 5, 6, 20),
    (1, '99', '17:00', '23:59', 'Mon-Fri', 3, 5, 9600, 16000, 21, 35, 3, 5, 8, 26),
    (1, '99', '00:00', '09:00', 'Mon-Fri', 3, 4, 4800, 9600, 10, 20, 3, 4, 5, 15),
    (1, '99', '00:00', '23:59', 'Sat,Sun', 3, 5, 11400, 17400, 24, 40, 3, 5, 12, 30);
```

| **rule id** | **profile id** | **prefix** | **start hour** | **end hour** | **days of the week** | **cpm warning** | **cpm critical** | **call duration warning** | **call duration critical** | **total calls warning** | **total calls critical** | **concurrent calls warning** | **concurrent calls critical** | **sequential calls warning** | **sequential calls critical** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | 99 | 09:00 | 17:00 | Mon-Fri | 3 | 5 | 7200 | 13200 | 16 | 35 | 3 | 5 | 6 | 20 |
| 2 | 1 | 99 | 17:00 | 23:59 | Mon-Fri | 3 | 5 | 9600 | 16000 | 21 | 35 | 3 | 5 | 8 | 26 |
| 3 | 1 | 99 | 00:00 | 09:00 | Mon-Fri | 3 | 4 | 4800 | 9600 | 10 | 20 | 3 | 4 | 5 | 15 |
| 4 | 1 | 99 | 00:00 | 23:59 | Sat,Sun | 3 | 5 | 11400 | 17400 | 24 | 40 | 3 | 5 | 12 | 30 |

**fraud_detection table example**

Data interpretation:

* **Rule 1** matches from **09:00 to 17:00**, **Monday to Friday**. It raises a
  **fraud warning** event and returns `-1` when a user either:
  * makes **between 3 and 4 calls within one minute**
  * makes one call lasting **between 7200 and 13199 seconds**
  * makes **between 16 and 34 calls** in the current interval
  * makes **between 3 and 4 simultaneous calls**
  * makes **between 6 and 19 consecutive calls** to the same destination
* The same **Rule 1** raises a **fraud critical** event and returns `-2` when a
  user either:
  * makes **5 or more calls within one minute**
  * makes one call lasting **13200 or more seconds**
  * makes **35 or more calls** in the current interval
  * makes **5 or more simultaneous calls**
  * makes **20 or more consecutive calls** to the same destination

### How does it work?

As stated in the [module documentation](/manual/devel/modules/fraud_detection), the
module monitors five parameters that are updated every time
[**check_fraud()**](/manual/devel/modules/fraud_detection#func_check_fraud) is called.
For each `(username, prefix)` tuple, the module keeps and monitors unique values
for those five parameters.

Each time a value changes, it is compared to the threshold values of the
matching rule. There are two threshold values for each parameter: warning and
critical. These thresholds, together with a destination prefix and a time
interval in which they apply, form a *rule*. A profile is a group of such
rules, selected through the `profile_id` argument of `check_fraud()`.

## Example script

First, load the dependencies and the **fraud_detection** module:

```opensips
loadmodule "db_mysql.so"
loadmodule "tm.so"
loadmodule "rr.so"
loadmodule "drouting.so"
loadmodule "dialog.so"
loadmodule "fraud_detection.so"
```

Set the database URLs. The **drouting** module is a dependency and must be able
to initialize normally, even though **fraud_detection** loads its own fraud
rules from the **fraud_detection** table.

```opensips
modparam("drouting", "db_url", "mysql://root:123456@localhost/opensips")
modparam("fraud_detection", "db_url", "mysql://root:123456@localhost/opensips")
```

Optionally, define `event_route` blocks in order to handle warning and critical
events inside the script. In current OpenSIPS versions, `event_route` support is
part of the core and does not require loading a separate module.

Assume that each initial INVITE should update the fraud statistics:

```opensips
route {
	if (has_totag()) {
		loose_route();
	} else if (is_method("INVITE")) {
		record_route();

		check_fraud($fU, $rU, 1);
		if ($rc < 0)
			xlog("[$ci] Fraud detection alert for user $fU, rc=$rc\n");
	}

	t_relay();
}
```

Some issues can be identified immediately through the return code. See the
module documentation for the full return code list. The last parameter of
`check_fraud()` is the **profile id** provisioned earlier.

Optionally, handle the warning and critical events:

```opensips
event_route [E_FRD_WARNING]
{
	xlog("E_FRD_WARNING: $params(param);$params(value);$params(threshold);$params(user);$params(called_number);$params(rule_id);$params(profile_id)\n");
}

event_route [E_FRD_CRITICAL]
{
	xlog("E_FRD_CRITICAL: $params(param);$params(value);$params(threshold);$params(user);$params(called_number);$params(rule_id);$params(profile_id)\n");
}
```

The event parameter names are `param`, `value`, `threshold`, `user`,
`called_number`, `rule_id` and `profile_id`.

## Legacy 2.1 Notes

OpenSIPS 2.1 used the same fraud rule table layout and the same
`check_fraud()` concept, but the surrounding tooling and event-route syntax
were different.

### Table creation

In OpenSIPS 2.1 deployments, table creation was commonly done through
`opensipsdbctl`. First make sure `opensipsctlrc` contains the desired SQL
engine, DB name and access credentials:

```bash
vim /usr/local/etc/opensips/opensipsctlrc
```

Then add the extra OpenSIPS tables:

```bash
opensipsdbctl extra
```

### Script example

The 2.1 script syntax typically passed string parameters to `check_fraud()`:

```opensips
route {
	if (has_totag()) {
		loose_route();
	} else {
		record_route();
		check_fraud("$fU", "$rU", "1");
		if ($rc < 0) {
			xlog("[$ci] Problems for user $fU rc=$rc\n");
		}
	}
	t_relay();
}
```

### Event parameters

OpenSIPS 2.1 examples used `fetch_event_params()` to copy event parameters into
script variables:

```opensips
event_route[E_FRD_WARNING] {
	fetch_event_params("$var(param);$var(val);$var(threshold);$var(user);$var(number);$var(ruleid)");
	xlog("E_FRD_WARNING: $var(param);$var(val);$var(threshold);$var(user);$var(number);$var(ruleid)\n");
}

event_route[E_FRD_CRITICAL] {
	fetch_event_params("$var(param);$var(val);$var(threshold);$var(user);$var(number);$var(ruleid)");
	xlog("E_FRD_CRITICAL: $var(param);$var(val);$var(threshold);$var(user);$var(number);$var(ruleid)\n");
}
```

For current OpenSIPS versions, use `$params(name)` as shown in the main example
above.
