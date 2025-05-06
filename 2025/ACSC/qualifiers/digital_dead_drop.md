# digital-dead-drop

## Description

```
Ever needed a simple way to leave messages for your local spies? Look no further, this digital dead drop is easy to use and even features a validation mechanism to check the source validity of messages.

Can you find a way to recover the admin's secret?
```

## Categories

- `crypto`

## Provided Files

- `digital-dead-drop.zip`

## Solve Process

I liked this challenge a lot because me being more of a programmer than CTF player, I haven't solved many crypto challenges yet. But this one seemed straight forward as I remembered, partly also from the very basic cryptography classes in school, what is happening here.

```py
# --- snip --- #
BLOCK_SIZE = Blowfish.block_size
TOKEN_LENGTH = 12
# --- snip --- #
def load_available_messages():
    with dbm.open(DBFILE, "c") as db:
        return [k.decode() for k in db.keys() if not k.startswith(b"_")]

def read_message(message_id):
    if message_id.startswith("_"):
        return None

    with dbm.open(DBFILE, "c") as db:
        if message_id not in db:
            return None
        data = db[message_id].decode()

    token = data[:TOKEN_LENGTH]
    message = data[TOKEN_LENGTH:]

    return message, token
    
def leave_message(message, token=None):
    if not token:
        token = generate_token()

    with dbm.open(DBFILE, "c") as db:
        while True:
            message_id = os.urandom(4).hex()
            if message_id not in db:
                break
        db[message_id] = (token + message).encode()
    
    return message_id, token
# --- snip --- #
def generate_token():
    return "".join(random.SystemRandom().choices(string.ascii_letters + string.digits, k=TOKEN_LENGTH))

def encrypt_token(token, key):
    iv = os.urandom(BLOCK_SIZE)
    cipher = Blowfish.new(key, Blowfish.MODE_CBC, iv)
    return iv + cipher.encrypt(pad(token.encode(), BLOCK_SIZE))

def decrypt_token(ciphertext, key):
    iv = ciphertext[:BLOCK_SIZE]
    cipher = Blowfish.new(key, Blowfish.MODE_CBC, iv)
    return unpad(cipher.decrypt(ciphertext[BLOCK_SIZE:]), BLOCK_SIZE)

def init_db():
    if not os.path.isfile(DBFILE + ".db"):
        key = os.urandom(32)
        flag_token = generate_token()
        flag_token_public = encrypt_token(flag_token, key).hex()
        leave_message("Hello, World!", token=flag_token)
        
        with dbm.open(DBFILE, "c") as db:
            db["_key"] = key
            db["_flag_token_public"] = flag_token_public.encode()
            db["_flag_token"] = flag_token.encode()

        return key, flag_token_public, flag_token
    else:
        with dbm.open(DBFILE, "c") as db:
            return db["_key"], db["_flag_token_public"].decode(), db["_flag_token"].decode()
# --- snip --- #
def main():
    key, flag_token_public, flag_token = init_db()
    print("Hello friend. Welcome to the Digital Dead Drop.")
    print(f"The administrator has left a message for you. Validate with {flag_token_public}")
    while True:
        print_menu()
        try:
            choice = int(input("> "))
            match choice:
                case 1:
                    message_id = input("Enter message ID: ")
                    message, token = read_message(message_id)
            
                    if message is None:
                        print("Message not found.")
                    else:
                        print(f"Message: {message}")
            
                case 2:
                    message = input("Enter message: ")
                    message_id, token = leave_message(message)
                    print(f"Message ID: {message_id}")
                    print(f"Token: {encrypt_token(token, key).hex()}")
            
                case 3:
                    messages = load_available_messages()
                    print("Messages:")
            
                    for message_id in messages:
                        print(f"  - {message_id}")
            
                case 4:
                    message_id = input("Enter message ID: ")
                    token_encr = bytes.fromhex(input("Enter token: "))
                    message, token = read_message(message_id)
            
                    if message is None:
                        print("Message not found.")
                    elif decrypt_token(token_encr, key) == token.encode():
                        print("Token is valid.")
                    else:
                        print("Token is invalid.")
                    
                case 5:
                    print("Goodbye!")
                    break
            
                case 1337:
                    # Super secret admin action validation method
                    token = input("Enter token: ")
                    if token == flag_token:
                        print(f"Good job, you found my secret!\n{FLAG}")
            
                case _:
                    print("Invalid choice.")
# --- snip --- #
```

This present us a very cool TUI for choosing between different menu options. The application starts by leaving a initial message with the token `flag_token`. Additionally, this token is presented to us at the start of the application as `flag_token_public`, which is `flag_token` encrypted with `key`.

In menu option 4, we can view messages, provided we know the correct key, which is then passed to `decrypt_token()`. The interesting part here is that this function does not really sanitize its error output, which means depending on the input, it will directly throw every error back to the TUI user. Since `unpad()` is directly passed to the `return` statement, which indicates a potential padding error.

This and the fact that blowfish's `MODE_CBC` is used, points us to a specific attack called a [Padding-Oracle Attack](). Simply put, this type of attack abuses the fact that we can probe a specific endpoint directly for potential padding errors, using arbitary input. We can abuse this to basically brute-force our way to the correct key.

I will link a great article by Mahmoud Jadaan which describes this in great detail [here](https://medium.com/@masjadaan/oracle-padding-attack-a61369993c86).

Armed with new knowledge and a suitable library (because we don't rewrite it if a good solution already exists) => [padding_oracle.py](https://github.com/djosix/padding_oracle.py), we can take on writing our solver script for this challenge.

```py
from pwn import log, process, remote
from padding_oracle import decrypt
from Crypto.Util.Padding import unpad
from Crypto.Cipher import Blowfish
from os import getenv

HOST = 'port.dyn.acsc.land'
PORT = 31624

if int(getenv("REMOTE", 0)) == 1:
    r = remote(HOST, PORT)
else:
    r = process("./challenge.py")

BLOCK_SIZE = Blowfish.block_size
FLAG_TOKEN_PUBLIC = ""
MESSAGE_ID = ""
DECRYPT_RESULT = ""

# get public token
output = r.recvuntil(b'> ')
FLAG_TOKEN_PUBLIC = bytes.fromhex(output.decode().split("\n")[1].strip().split(" ")[-1])
log.info(f"Public Token: {FLAG_TOKEN_PUBLIC}")

# get message id
r.sendline(b'3')
output = r.recvuntil(b'> ')
MESSAGE_ID = output.decode().split("\n")[1].strip().split("-")[1].strip()
log.info(f"Message ID: {MESSAGE_ID}")

# verify message
def verify_message(ciphertext: bytes) -> bool:
    CIPHER_HEX = ciphertext.hex()
    r.sendline(b'4')
    r.recvuntil(b': ')
    r.sendline(MESSAGE_ID.encode())
    r.recvuntil(b': ')
    r.sendline(CIPHER_HEX.encode())
    output = r.recvuntil(b'> ')
    DECRYPT_RESULT = output.decode(errors="ignore").split("\n")[0]
    if "Error:" in DECRYPT_RESULT:
        return False
    else:
        return True

verify_message(FLAG_TOKEN_PUBLIC)

def oracle(ciphertext: bytes):
    RESULT = verify_message(ciphertext)
    if RESULT:
        return True
    elif not RESULT:
        return False
    else:
        raise RuntimeError('Unexpected behavior')

assert len(FLAG_TOKEN_PUBLIC) % BLOCK_SIZE == 0

plaintext = decrypt(
    FLAG_TOKEN_PUBLIC,
    block_size=BLOCK_SIZE,
    oracle=oracle,
    num_threads=1
)

FLAG_TOKEN = unpad(plaintext, BLOCK_SIZE).decode()
log.info(f"Plaintext recovered: {FLAG_TOKEN}")

# get the flag
r.sendline(b'1337')
output = r.recvuntil(b': ')
r.sendline(FLAG_TOKEN.encode())
output = r.recvuntil(b'> ')
FLAG = output.decode().split("\n")[1]
log.success(f"Flag: {FLAG}")
```

And while this will take a while on the remote target, it yields us another flag eventually.
