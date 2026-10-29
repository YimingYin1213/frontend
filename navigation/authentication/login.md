---
layout: page
title: Login
permalink: /login
search_exclude: true
show_reading_time: false
safe_passage: true
safe_passage_auth: true
hide_title: true
---
<main class="auth-main" id="main-content">
    <div class="auth-intro">
        <p class="eyebrow">A calm place to begin again</p>
        <h1>Welcome to your <em>next step.</em></h1>
        <p>Sign in to continue, or create an account to get started with Safe Passage Heals.</p>
    </div>
    <div class="auth-columns">
        <section class="auth-card auth-login-card" aria-labelledby="pythonTitle">
            <div class="card-heading"><div><h2 id="pythonTitle">Welcome back</h2><p>We’re glad you’re here. Sign in to continue.</p></div><span class="card-icon" aria-hidden="true">♡</span></div>
            <form id="pythonForm" onsubmit="loginBoth(); return false;">
                <div class="form-group"><label for="uid">Username</label><input type="text" id="uid" name="uid" placeholder="Enter your username" autocomplete="username" required></div>
                <div class="form-group"><label for="password">Password</label><input type="password" id="password" name="password" placeholder="Enter your password" autocomplete="current-password" required></div>
                <button type="submit" class="ocs__btn large pill safe-passage-submit">Sign in <span aria-hidden="true">→</span></button>
                <p id="message" class="form-message" role="status" aria-live="polite"></p>
            </form>
            <div class="security-note"><span aria-hidden="true">✳</span>Your information is used to support your account and program experience.</div>
        </section>
        <section class="auth-card auth-signup-card" aria-labelledby="signupTitle">
            <div class="card-heading"><div><h2 id="signupTitle">Start your journey</h2><p>Create an account to get connected.</p></div><span class="card-icon" aria-hidden="true">✦</span></div>
            <form id="signupForm" onsubmit="signup(); return false;">
                <div class="form-grid">
                    <div class="form-group"><label for="name">Name</label><input type="text" id="name" name="name" placeholder="Your name" autocomplete="name" required></div>
                    <div class="form-group"><label for="signupUid">Username</label><input type="text" id="signupUid" name="signupUid" placeholder="Choose a username" autocomplete="username" required></div>
                    <div class="form-group"><label for="signupEmail">Email</label><input type="email" id="signupEmail" name="signupEmail" placeholder="you@example.com" autocomplete="email" required></div>
                </div>
                <div class="form-group"><label for="signupPassword">Create password</label><input type="password" id="signupPassword" name="signupPassword" placeholder="Create a password" autocomplete="new-password" required></div>
                <button type="submit" class="ocs__btn large pill safe-passage-submit">Create account <span aria-hidden="true">→</span></button>
                <p id="signupMessage" class="form-message signup-message" role="status" aria-live="polite"></p>
            </form>
        </section>
    </div>
</main>
<script type="module">
    import { login, pythonURI, javaURI, fetchOptions } from '{{site.baseurl}}/assets/js/api/config.js';
    // Function to handle both Python and Java login simultaneously
    window.loginBoth = function () {
    javaLogin();  // Call Java login
    pythonLogin();
};
    // Function to handle Python login
    window.pythonLogin = function () {
        const options = {
            URL: `${pythonURI}/api/authenticate`,
            callback: pythonDatabase,
            message: "message",
            method: "POST",
            cache: "no-cache",
            body: {
                uid: document.getElementById("uid").value,
                password: document.getElementById("password").value,
            }
        };
        login(options);
    }
    // Function to handle Java login
    window.javaLogin = function () {
    const loginURL = `${javaURI}/authenticate`;
    const databaseURL = `${javaURI}/api/person/get`;
    const signupURL = `${javaURI}/api/person/create`;
    const userCredentials = JSON.stringify({
        uid: document.getElementById("uid").value,
        password: document.getElementById("password").value,
    });
    const loginOptions = {
        ...fetchOptions,
        method: "POST",
        body: userCredentials,
    };
    console.log("Attempting Java login...");
    fetch(loginURL, loginOptions)
        .then(response => {
            if (!response.ok) {
                throw new Error("Invalid login");
            }
            return response.json();
        })
        .then(data => {
            console.log("Login successful!", data);
            window.location.href = '{{site.baseurl}}/profile';
            // Fetch database after login success using fetchOptions
            return fetch(databaseURL, fetchOptions);
        })
        .then(response => {
            if (!response.ok) {
                throw new Error(`Spring server response: ${response.status}`);
            }
            return response.json();
        })
        .then(data => {
            console.log("Java database response:", data);
        })
        .catch(error => {
            console.error("Login failed:", error.message);
            // If login fails, attempt account creation
            if (error.message === "Invalid login") {
                alert("Login for Spring failed. Creating a new Java account...");
                const signupData = JSON.stringify({
                    uid: document.getElementById("uid").value,
                    email: document.getElementById("uid").value + "@gmail.com",
                    dob: "11-01-2024", // Static date, can be modified
                    name: document.getElementById("uid").value,
                    password: document.getElementById("password").value,
                    kasmServerNeeded: false,
                });
                const signupOptions = {
                    ...fetchOptions,
                    method: "POST",
                    body: signupData,
                };
                fetch(signupURL, signupOptions)
                    .then(signupResponse => {
                        if (!signupResponse.ok) {
                            throw new Error("Account creation failed!");
                        }
                        return signupResponse.json();
                    })
                    .then(signupResult => {
                        console.log("Account creation successful!", signupResult);
                        alert("Account Creation Successful. Logging you into Flask/Spring!");
                        // Retry login after account creation
                        return fetch(loginURL, loginOptions);
                    })
                    .then(newLoginResponse => {
                        if (!newLoginResponse.ok) {
                            throw new Error("Login failed after account creation");
                        }
                        console.log("Login successful after account creation!");
                        // Fetch database after successful login
                        return fetch(databaseURL, fetchOptions);
                    })
                    .then(response => {
                        if (!response.ok) {
                            throw new Error(`Spring server response: ${response.status}`);
                        }
                        return response.json();
                    })
                    .then(data => {
                        console.log("Java database response:", data);
                    })
                    .catch(newLoginError => {
                        console.error("Error after account creation:", newLoginError.message);
                    });
            } else {
                console.log("Logged in!");
            }
        });
};
    // Function to fetch and display Python data
    function pythonDatabase() {
        const URL = `${pythonURI}/api/id`;
        fetch(URL, fetchOptions)
            .then(response => {
                if (!response.ok) {
                    throw new Error(`Flask server response: ${response.status}`);
                }
                return response.json();
            })
            .then(data => {
                window.location.href = '{{site.baseurl}}/profile';
            })
            .catch(error => {
                document.getElementById("message").textContent = `Error: ${error.message}`;
            });
    }
    window.signup = function () {
        const signupButton = document.querySelector(".auth-signup-card button");
        const signupMessage = document.getElementById("signupMessage");
        signupButton.disabled = true;
        signupButton.classList.add("disabled");
        fetch(`${pythonURI}/api/user`, {
            ...fetchOptions,
            method: "POST",
            body: JSON.stringify({
                name: document.getElementById("name").value,
                uid: document.getElementById("signupUid").value,
                email: document.getElementById("signupEmail").value,
                password: document.getElementById("signupPassword").value,
            })
        })
            .then(response => {
                if (!response.ok) throw new Error(`Signup failed (${response.status})`);
                return response.json();
            })
            .then(data => {
                if (data.success === false) throw new Error(data.message || "Signup failed");
                signupMessage.textContent = "Signup successful! You can now sign in.";
            })
            .catch(error => {
                console.error("Signup Error:", error);
                signupMessage.textContent = `Signup Error: ${error.message}`;
                signupButton.disabled = false;
                signupButton.classList.remove("disabled");
            });
    }
    function javaDatabase() {
        const URL = `${javaURI}/api/person/get`;
        fetch(URL, fetchOptions)
            .then(response => {
                if (!response.ok) {
                    throw new Error(`Spring server response: ${response.status}`);
                }
                return response.json();
            })
            .catch(error => {
                console.error("Java Database Error:", error);
            });
    }
</script>
