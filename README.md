## Extend Auth

This is an app to allow cross site cookies for given domains.

Update site configs to have cross sites list
```js
"allowed_cross_sites": [
    "http://localhost:5173"
]
```

Adding these lines of code to your own custom-app will also work
```js
import frappe

def get_allowed_cross_sites():
    """Fetch allowed cross-origin domains from site_config.json"""
    return frappe.get_conf().get("allowed_cross_sites", [])

def patch_flush_cookies():
    """Patch Frappe's flush_cookies method to enforce SameSite=None and Secure."""
    if not hasattr(frappe.local, "cookie_manager"):
        return  # Ensure frappe.local is initialized

    if hasattr(frappe.local.cookie_manager, "_patched_flush"):
        return  # Avoid multiple patches

    original_flush_cookies = frappe.local.cookie_manager.flush_cookies

    def patched_flush_cookies(self, response):
        """Modify cookies before they are sent to the client."""
        print("🚀 Checking request origin for cookie patching...")
        
        # Get request origin
        request = getattr(frappe.local, "request", None)
        origin = request.headers.get("Origin") if request else None

        # Fetch allowed cross-origin domains
        allowed_sites = get_allowed_cross_sites()
        if origin and origin in allowed_sites:
            print(f"✅ Origin {origin} is allowed, applying SameSite=None")
            for key, opts in self.cookies.items():
                opts["samesite"] = "None"
                opts["secure"] = True  # Required for SameSite=None
        else:
            print(f"❌ Origin {origin} is not allowed, skipping SameSite=None")
        # Call the original flush_cookies method

        return original_flush_cookies( response)

    # Bind the patched function to the cookie_manager instance
    from types import MethodType
    frappe.local.cookie_manager.flush_cookies = MethodType(patched_flush_cookies, frappe.local.cookie_manager)
    frappe.local.cookie_manager._patched_flush = True  # Mark as patched

# Hook into the request lifecycle
def before_request_handler():
    print("Running before request handler...")
    patch_flush_cookies()

# Register the hook
before_request = [before_request_handler]
```
#### License

mit