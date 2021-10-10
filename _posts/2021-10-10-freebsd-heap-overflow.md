---
layout: post
title: "IEEE80211_IOC_MLME kernel heap overflow"
date: 2021-10-10 13:37:41 +0100
categories: kernel FreeBSD
---
**What is IEEE80211_IOC_MLME?**

`ieee80211_ioctl_setmlme()` / `IEEE80211_IOC_MLME` is a FreeBSD ioctl reachable by root, and it's used to control the MLME state machine, anyway this isn't required to understand the bug.

**Where's the bug?**

Looking at `ieee80211_ioctl_setmlme()` we can see 2 things:

- mlme struct is initalized with user controlled data (`ireq->i_data`)
- different functions are called if the `op` / `opmode` field is set to some specific value

If we set iv_opmode field to `IEEE80211_M_IBSS` / `IEEE80211_M_AHDEMO` and im_op field to `IEEE80211_MLME_ASSOC`, `setmlme_assoc_adhoc()` is called with 3 untrusted user inputs, the interesting one is `im_ssid_len`, which is a `uint8_t` given by the user, and it's passed as `ssid_len`.

If we look at the function we can notice two interesting things:

- `ssid_len` check just makes sure that it isn't 0
- our untrusted `ssid_len` value is passed to some `memcpy()`

Now the question is: are these `memcpy()` interesting? Well one is:

```
memcpy(sr->sr_ssid[0].ssid, ssid, ssid_len);
```

`sr->sr_ssid[0].ssid` is a 32 bytes buffer from this struct:

```c
struct {
	int	 len;				/* length in bytes */
	uint8_t ssid[IEEE80211_NWID_LEN];	/* ssid contents */
} sr_ssid[IEEE80211_IOC_SCAN_MAX_SSID];
```

`IEEE80211_NWID_LEN` is 32, and `IEEE80211_IOC_SCAN_MAX_SSID` is 3, so you can write up to 151 bytes out-of-bounds.

**What about the exploitability of the bug?**

The size of the overflow is quite big, but from static analysis seems like the data which are gonna be written after the buffer aren't controlled, but taken somewhere from the stack (with some debugging it should be quite easy to figure out what is gonna be copied), even if random data are copied it could still be possible to exploit this issue, for example by overwriting a reference counter until it reaches 0 (since most of the times you can check if this condition is reached).

**How the bug was patched?**

The patch for the bug is straightforward, they just added a size check to ensure that the user cannot provide a size bigger than the `ssid` buffer:

```c
if (ssid_len == 0 || ssid_len > sizeof(ssid))
 	return EINVAL;
```

