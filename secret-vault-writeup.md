## 2025-07-secret-vault

###### tags:`CodeHawks First Flights`

### ✅ Root + Impact: Improper Access Control in `get_secret` Function Leads to Unauthorized Secret Disclosure

#### Description
The `get_secret()` function attempts to restrict reads by asserting `caller == @owner` and then always borrowing the Vault at the named address @owner. However:
- The only check is comparing a user‑supplied argument (`caller`) to a compile‑time constant (`@owner`). Any external caller can just pass `@owner` and the check succeeds
- In Aptos, view functions have no signer context. There is no authenticated `&signer` to bind the request to the real caller, therefore this pattern cannot enforce access control

#### Risk

**Likelihood**: High
- The `get_secret()` function is publicly callable and `@owner` is embedded in bytecode, which is discoverable
- Attackers can trivially pass the expected address and retrieve the secret


**Impact**: High
- Full confidentiality breach of the owner’s secret. If the secret controls off‑chain privileges or is sensitive data, the impact extends beyond on‑chain state. Additionally, inability to rotate the secret (due to move_to only) exacerbates exposure once leaked


#### Proof of Concept
Add the following test, then run the command: `aptos move test -f test_anyone_can_read_secret`
```move
#[test(owner = @0xcc, attacker = @0xdead)]
fun test_anyone_can_read_secret(owner: &signer, attacker: &signer) acquires Vault {
    use aptos_framework::account;

    account::create_account_for_test(signer::address_of(owner));
    account::create_account_for_test(signer::address_of(attacker));

    // Owner stores a secret once.
    set_secret(owner, b"i'm a secret");

    // Attacker learns owner address (on-chain/public) and calls the view:
    let leaked: String = get_secret(signer::address_of(owner));

    // Succeeds: attacker recovered the secret.
    assert!(leaked == utf8(b"i'm a secret"), 0);

    let msg = utf8(b"retrieved secret: ");
    string::append(&mut msg, leaked);
    debug::print(&msg);
}
```

PoC Results:

```
aptos move test -f test_anyone_can_read_secret
INCLUDING DEPENDENCY AptosFramework
INCLUDING DEPENDENCY AptosStdlib
INCLUDING DEPENDENCY MoveStdlib
BUILDING aptos-secret-vault
Running Move unit tests
[debug] "retrieved secret: i'm a secret"
[ PASS    ] 0x234::vault::test_anyone_can_read_secret
Test result: OK. Total tests: 1; passed: 1; failed: 0
{
  "Result": "Success"
}
```

#### Recommended Mitigation
For proper access control, one way is to simly remove `#[view]` and require an authenticated signer:

```move
public fun get_secret_auth(caller: &signer): String acquires Vault {
    let addr = signer::address_of(caller);
    let v = borrow_global<Vault>(addr);
    // Assuming String has `copy` ability in your toolchain:
    v.secret
}
```

- Note that this method only prevent others from calling `get_secret()`, but it does not provide confidentiality, since any on‑chain plaintext is publicly readable
- To provide confidentiality, consider storing ciphertext on-chain and decrypt off-chain

### ✅ Root + Impact: Write-once / Secret Rotation DoS

#### Description
- The `set_secret()` function uses `move_to` to publish a new Vault resource under the caller’s account every time it is invoked
- In Move, `move_to` aborts if the resource already exists at that address, which means:
    - The first call works, storing the initial secret.
    - Any subsequent call by the same account aborts, because Vault already exists

```move
public entry fun set_secret(caller:&signer,secret:vector<u8>){
    let secret_vault = Vault{secret: string::utf8(secret)};
    move_to(caller,secret_vault);
    event::emit(SetNewSecret {});
}
```
#### Risk

**Likelihood**: High
- Any real user will eventually need to update or change the stored secret. The bug is guaranteed to manifest on the second use.

**Impact**: Medium
- Permanent denial of service on secret rotation.
- Breaks usability and may lock users into an outdated or incorrect secret forever.


#### Proof of Concept
Add the following test, then run the command: `aptos move test -f test_move_to_twice_aborts`

```move
#[test(owner = @0xcc)]
#[expected_failure] // second move_to<Vault> at same address aborts (resource already exists)
fun test_move_to_twice_aborts(owner: &signer) {
    use aptos_framework::account;

    account::create_account_for_test(signer::address_of(owner));

    // First publish succeeds.
    set_secret(owner, b"first");

    // Second publish tries move_to again → aborts because a Vault already exists.
    // This shows "one-shot" behavior: you can't rotate/update via move_to.
    set_secret(owner, b"second");
}
```

#### Recommended Mitigation

Replace unconditional `move_to` with a create-or-update pattern, this allows the first write to succeed and subsequent calls to safely update the secret.

```move
public entry fun set_secret_rotatable(caller:&signer,secret:vector<u8>) acquires Vault {
    if (exists<Vault>(signer::address_of(caller))) {
        let v = borrow_global_mut<Vault>(signer::address_of(caller));
        v.secret = string::utf8(secret);
    } else {
        move_to(caller, Vault { secret: string::utf8(secret) });
    };
    event::emit(SetNewSecret {});
}
```

### ✅ Root + Impact: Read/Write Location Mismatch Risk

#### Description
According to the project's documentation, only the owner may set and retrieve their secret. However, there exists mismatch between `set_secret()` and `get_secret()`:
- `set_secret()` writes the Vault resource under the caller’s address
- `get_secret` reads the Vault from a hardcoded address `@owner`

This result in read DoS if fixed `@owner` is differ from the caller of `set_secret`

```move
public entry fun set_secret(caller:&signer,secret:vector<u8>){
    let secret_vault = Vault{secret: string::utf8(secret)};
@>  Write the Vault resource under the caller's address 
    move_to(caller,secret_vault);
    event::emit(SetNewSecret {});
}

#[view]
public fun  get_secret (caller: address):String acquires Vault{
    assert! (caller == @owner,NOT_OWNER);
@>  Reads the Vault from a hardcoded address @owner
    let vault = borrow_global<Vault >(@owner);
    vault.secret
}
```
 

#### Risk

**Likelihood**: High
- unless the caller’s address exactly matches the bound `@owner`, it fails

**Impact**: Medium
- Mismatch between storage and retrieval
- Secrets may appear “missing” even though they were stored


#### Proof of Concept

Add the following test, then run the command: `aptos move test -f test_read_write_mismatch`

```move
#[test(owner = @0xcc, user = @0x123)]
#[expected_failure]
fun test_read_write_mismatch(owner: &signer, user: &signer) acquires Vault {
    use aptos_framework::account;

    account::create_account_for_test(signer::address_of(owner));
    account::create_account_for_test(signer::address_of(user));

    // User stores a secret once.
    set_secret(user, b"i'm a secret");
    // User fails to read their own secret.
    let leaked: String = get_secret(signer::address_of(user));
}
```

#### Recommended Mitigation
Align read and write to the same address authority:
- **Single-owner design**: Enforce `signer::address_of(caller) == @owner` inside `set_secret`, and always write/read from @owner. This guarantees consistent access for the designated owner.
- **Multi-user design**: Remove the hardcoded `@owner`. Instead, use `signer::address_of(caller)` consistently for both writing and reading, and consider adding an `owner` field inside the Vault resource for explicit ownership tracking.
