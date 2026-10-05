

# Security Review Accent GeoGuessr

## Application Status

The publicly reachable instance of the application at accentgeoguessr-classroom.streamlit.app has currently been taken down temporarily. This allows the measures described in this report to be implemented without exposing users to an active risk during the cleanup. The findings in this report reflect the state before the shutdown. Once the corrections are in place, the application can be republished with improved security.

## Scope of the Review

The application from the public GitHub repository github.com/quentinrauschenbach/accent_geoguessr was examined. The entire code contained in the repository was reviewed, with a particular focus on the two main files app.py and app_local.py, since they contain the complete application logic. In addition, the dependencies in requirements.txt and the commit history were considered. As a complement, the live instance was visited manually and checked in both areas of the application: the normal student view at https://accentgeoguessr-classroom.streamlit.app/ and the teacher panel at https://accentgeoguessr-classroom.streamlit.app/?role=teacher. The review covered the server responses, the cookies being set, and the visible configuration of both areas.

## 1. Summary

The application keeps the entire game state in a single global store on the server, and authentication exists practically only for the teacher, implemented weakly. For a pure classroom game most of the findings would be tolerable. However, the instance ran publicly on the internet, which makes the critical findings realistically exploitable. The main risk does not lie in classic injection vulnerabilities but in logic and configuration: takeover of the teacher role, manipulation of the game state, and phishing of students through a manipulated QR code. In addition, weaknesses were observed on the delivery layer while inspecting the server responses.

## 2. Critical Findings

### 2.1 Password in Plaintext in a Public Repository

In app.py the teacher password is readable line by line:

```python
TEACHER_PASSWORD = "0712"
```

The password is publicly visible. Anyone who knows the repository name can take over the teacher function immediately. It also consists of four digits that look very much like a date, so even a change remains easy to guess. Merely changing the file is not enough, because the old value remains permanently preserved in the commit history. Tools such as BFG Repo Cleaner or git filter repo are suitable for cleaning the history. In the simplest case the old value is considered permanently burned and never used again.

Recommendation: Move the password into Streamlit Secrets, loading it via st.secrets. The file secrets.toml belongs in the gitignore. The new password should contain at least twelve random characters without a recognizable pattern.

### 2.2 No Protection Against Password Guessing

The teacher login compares the password directly on every attempt, without a counter, without delay, without lockout. A four digit code comprises ten thousand combinations and can be exhausted by a simple script in under a minute. Streamlit does not limit requests by default.

Recommendation: An attempt counter per session and address, with a multi minute lockout after five failed attempts. More robust would be a real gate in front of the app, such as a reverse proxy with basic auth on the teacher route. Secrets plus rate limiting are nevertheless sufficient for this purpose.

## 3. High Risks

### 3.1 Manipulable Join URL Through the Host Header

```python
host_url = st.context.headers.get("host", "localhost:8501")
STUDENT_JOIN_URL = f"https://{host_url}"
```

The Host header is sent by the client and is therefore attacker controlled. If someone opens the page with a forged Host header, the app generates a QR code pointing to the attacker's domain. If that code is displayed on the projector, students land on a cloned page. This scenario is known as web cache poisoning or host header injection.

Recommendation: Store the public URL statically, for example as a Streamlit secret, instead of deriving it from the request. Alternatively use a whitelist of allowed hostnames and reject deviations.

### 3.2 Manipulable Game State

The entire game state sits in a single dictionary shared via st.cache_resource. Three problems result, each harmless alone but substantial together.

First, nicknames are not protected. Anyone entering the same name as a classmate writes entries under that identity into the all_guesses table, because the name alone serves as the key. This allows foreign results to be spoiled or foreign points to be claimed.

Second, the guess lock exists only in session_state. The LOCK IN GUESS button sets a flag in the browser session. Opening a fresh incognito tab makes the flag disappear and allows guessing again. Nothing on the server prevents multiple guesses per person and round.

Third, no deadline exists. Guesses still flow into the scoring while the reveal is already visible on the projector, as long as the round is still active in the counter. Whoever reads the true position from the screen and quickly opens a fresh tab receives the full points.

Recommendation: A random join token per student, stored with the name and sent with every guess. On the server, check whether an entry for the combination of round and name already exists and discard additional ones. Once show_leaderboard is true, stop accepting guesses for the current round. Each of these measures takes only a few lines.

### 3.3 Role Check Presumably Only at the Entrance

Access to the teacher view runs through the query parameter role=teacher and the password. From the visible code it could not be fully determined whether authentication is rechecked before every teacher block or only on first load. A common Streamlit mistake is that after a rerun or a manipulated session_state protected areas become reachable without rechecking. It should therefore be verified whether actions such as upload, round changes, or resets depend only on the teacher code block executing, and not on an explicitly checked variable such as st.session_state.authenticated.

Recommendation: A single variable is_teacher stored firmly in session_state, set only after a successful password comparison, and every teacher block begins by checking that variable.

## 4. Medium Risks

### 4.1 File Upload Without Visible Hardening

The upload of audio and video files into the clips folder shows no validation of filename, size, or content type in the reviewed code. Three concrete dangers: A filename containing path components can land outside the target folder if the name is used unfiltered. Without a size limit a single upload can fill the free storage of Streamlit Cloud, which is about one gigabyte, after which the application becomes unreachable for everyone. Without extension checks arbitrary files land on the server.

Recommendation: Regenerate the stored filename server side, for example with uuid, keeping the original name only as display text. A hard limit of twenty to fifty megabytes. Restrict extensions strictly to mp3, m4a, wav, and mp4, ideally also checking the mime type. Offer the upload exclusively behind teacher authentication.

### 4.2 Global Control Functions Reachable by Students

Even though the interface shows control functions only to the teacher, this is purely a display concern. Streamlit applications have no server side role concept; every session executes the same code. If actions such as round changes or game resets run depending on the display path instead of authentication, a technically savvy student can trigger them. This check belongs to the same effort as point 3.3.

## 5. Low Risks

The file requirements.txt contains no version pins. Acceptable for a classroom project, but pinning prevents unpleasant surprises from Streamlit updates.

Nicknames have no length limit. Very long names distort the leaderboard; thirty characters plus a restriction to sensible characters is a one line measure.

app_local.py generates the QR code with the local address over unencrypted http. Tolerable in a protected school network, but everyone on the same network sees the traffic and can join the game. The variant should be disabled for operation outside the classroom.

The game memory lives only in RAM; a restart deletes everything. This is not a security issue but explains the absence of server side safeguards, and would be an argument for moving the state to a sqlite file or small database if needed.

## 6. Prioritized Remediation Order

First comes the password. Use secrets, choose a strong password, treat the old value as permanently burned, and clean the history if possible.

Next, harden authentication: rate limiting on login and a consistently checked is_teacher variable before every teacher block, including upload and reset.

Then store the join URL statically instead of building it from the host header.

After that, game integrity: join tokens, duplicate checks per round and name, and a guess cutoff from the moment of the reveal.

Then harden the upload with regenerated filenames, a size limit, and extension checks.

Finally the smaller items: version pinning, name length limits, and awareness of plaintext operation on the local network.

## 7. Observations on the Delivery Layer

Besides the source code, the live instance was visited, in particular the responses the server sends to the browser and the cookies set in both areas of the application, the student panel and the teacher panel. This layer lies outside the application's control on Streamlit Community Cloud, so many of the following points should be understood as documented and platform side.

### 7.1 Content Security Policy, Not Present

No Content Security Policy was observed. A CSP is the most important line of defense against the injection of foreign scripts, in other words against cross site scripting. In a Streamlit application this is especially relevant because the interface consists of dynamically loaded components, and an attacker who succeeds in injecting content would have free rein without a CSP.

Recommendation: Set a Content Security Policy via the header of the same name. A sensible start restricts default-src to self and explicitly allows only the sources the application actually needs, namely the OpenStreetMap map tiles and the Streamlit specific connections. Start in report only mode, watch the console for violations, and tighten the policy afterwards.

### 7.2 X-Frame-Options and Clickjacking, Not Present

The page can be embedded in a foreign frame, so frame protection is missing. This enables clickjacking, the overlaying of the real page with a deceptive surface through which a user unknowingly triggers actions. In the school context the risk is low, but the fix costs almost nothing.

Recommendation: Either set the classic header X-Frame-Options with the value DENY or SAMEORIGIN, or more modern the directive frame-ancestors none inside the Content Security Policy from point 7.1.

### 7.3 Redirection and Missing Strict Transport Security

The first redirect from http to https goes to a different host. As a result the Strict Transport Security header is discarded on first contact, because HSTS is only valid over an encrypted connection to the same host. It was additionally observed that the instance's responses contain no Strict Transport Security header at all.

Recommendation: Rebuild the chain so the first hop to https happens on the same domain, with any further redirects afterwards. Little influence exists on Streamlit Community Cloud; the path leads through an own domain with a proxy in front.

### 7.4 Subresource Integrity, Not Present

External scripts are loaded encrypted but without integrity verification. SRI means the HTML attribute carries a hash of the expected script and the browser rejects the file if it has been altered. The practical relevance here is low because everything is at least transferred encrypted.

Recommendation: Add the attributes integrity and crossorigin to all external script and link tags. Since a Streamlit application does not write these tags itself, the CSP from point 7.1 helps most here.

### 7.5 Referrer Policy, Not Set

The header is missing from the observed responses. When clicking external links the full target address including any parameters can thereby be transmitted to the foreign page.

Recommendation: Set the Referrer-Policy header to strict-origin-when-cross-origin.

### 7.6 Cross Origin Policies, Not Set

The three headers Cross Origin Embedder Policy, Cross Origin Opener Policy, and Cross Origin Resource Policy are missing. Among other things they prevent foreign pages from building window relationships to the application or embedding its resources.

Recommendation: same-origin for the opener policy, require-corp or credentialless for the embedder policy, and same-origin for the resource policy. Test the settings in observation mode first, because the folium maps load external tiles.

### 7.7 Cookies, a Mixed Picture

The Streamlit own session cookies are correctly protected with Secure, HttpOnly, and SameSite. Notable in contrast is a cookie named proxy-tracking-id set without the HttpOnly and without the Secure flag. It originates from the platform infrastructure and cannot be controlled by the application itself. Since this cookie serves connection tracking rather than carrying a real session, the concrete damage is manageable. The combination with the missing Content Security Policy from point 7.1 is nevertheless uncomfortable, because a missing CSP lowers the injection threshold while an exposed cookie exists. The remedy is the same as there: an own domain with a proxy in front where all set headers are controlled directly.

### 7.8 Further Positive Observations

There are no open CORS allowances, and the main page contains the nosniff header, which prevents mime type sniffing. Notably this header is missing on the static asset routes under build and assets. The risk is low. With a later move behind an own proxy, the header belongs on every response, most simply via a global proxy configuration.

### 7.9 Server Information

Inspecting the connection data revealed that Google Cloud, a content delivery network, and an Nginx sit behind the application, with the Nginx version number visible. Such information allows an attacker to search specifically for vulnerabilities of exactly that version. Since the server infrastructure on Streamlit Community Cloud is not controllable, the point remains as a documented note.

### 7.10 What Was Not Observed

No open debug endpoints, no directory listings, no visible error messages with internal details, no passwords transmitted unencrypted, and no private keys in the responses were noticed. The relevant attack surfaces of this application are not the classic web vulnerabilities anyway, because there is no database and no server side query translating user input into SQL or shell commands. A deep technical assessment becomes sensible only once a database or real token based authentication is added.

## 8. Overall Assessment

The picture is twofold. Within the application code the critical topics are the password, rate limiting, and state manipulation. On the delivery layer the Content Security Policy and frame protection are missing above all, and a platform tracking cookie is insufficiently protected. Following the password and authentication, the third measure is the Content Security Policy together with frame-ancestors, then the statically stored join URL, game integrity, and the upload. The codebase is solidly structured for a weekend project created with AI assistance. The temporary shutdown of the instance is the right step to implement the corrections calmly before the application becomes publicly reachable again.
