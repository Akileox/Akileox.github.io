---
layout: default
permalink: /before-airforce/
title: "입대 전 준비"
author_profile: false
classes: wide
sitemap: false
---

<div class="gate-wrap" markdown="0">
  <div class="gate-card">
    <svg class="gate-card__icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="11" width="18" height="10" rx="2"></rect><path d="M7 11V7a5 5 0 0 1 10 0v4"></path></svg>
    <p class="gate-card__title">입대 전 준비</p>
    <p class="gate-card__desc">개인적으로 정리한 페이지입니다.<br>6자리 코드를 입력하면 이동합니다.</p>
    <form class="gate-form" id="gate-form" autocomplete="off">
      <input class="gate-input" id="gate-input" type="text" inputmode="numeric" pattern="[0-9]*" maxlength="6" placeholder="••••••" aria-label="6자리 코드">
      <p class="gate-error" id="gate-error"></p>
      <button class="gate-btn" type="submit" id="gate-submit">확인</button>
    </form>
    <p class="gate-hint">코드를 모르면 이 페이지는 볼 수 없습니다.</p>
  </div>
</div>

<script>
(function () {
  var ENC = {
    "salt": "ihiN2xqxvJPG/ptwEqQHaA==",
    "iv": "0QsfWdazwg6URqHS",
    "ciphertext": "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",
    "iterations": 200000
  };

  function b64ToBytes(b64) {
    var bin = atob(b64);
    var bytes = new Uint8Array(bin.length);
    for (var i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i);
    return bytes;
  }

  var form = document.getElementById("gate-form");
  var input = document.getElementById("gate-input");
  var errorEl = document.getElementById("gate-error");
  var submitBtn = document.getElementById("gate-submit");

  input.addEventListener("input", function () {
    input.value = input.value.replace(/[^0-9]/g, "").slice(0, 6);
    input.classList.remove("gate-input--error");
    errorEl.textContent = "";
  });

  form.addEventListener("submit", async function (e) {
    e.preventDefault();
    var code = input.value.trim();
    if (code.length !== 6) {
      errorEl.textContent = "6자리 숫자를 입력하세요.";
      input.classList.add("gate-input--error");
      return;
    }
    if (!window.crypto || !window.crypto.subtle) {
      errorEl.textContent = "이 브라우저는 지원하지 않습니다.";
      return;
    }
    submitBtn.disabled = true;
    try {
      var enc = new TextEncoder();
      var baseKey = await crypto.subtle.importKey("raw", enc.encode(code), { name: "PBKDF2" }, false, ["deriveKey"]);
      var key = await crypto.subtle.deriveKey(
        { name: "PBKDF2", salt: b64ToBytes(ENC.salt), iterations: ENC.iterations, hash: "SHA-256" },
        baseKey,
        { name: "AES-GCM", length: 256 },
        false,
        ["decrypt"]
      );
      var plainBuf = await crypto.subtle.decrypt(
        { name: "AES-GCM", iv: b64ToBytes(ENC.iv) },
        key,
        b64ToBytes(ENC.ciphertext)
      );
      var html = new TextDecoder().decode(plainBuf);
      sessionStorage.setItem("baf-content", html);
      window.location.href = "/before-airforce/notes/";
    } catch (err) {
      errorEl.textContent = "코드가 틀렸습니다.";
      input.classList.add("gate-input--error");
      input.select();
    } finally {
      submitBtn.disabled = false;
    }
  });
})();
</script>
