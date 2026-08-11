---
layout: default
permalink: /before-airforce/notes/
title: "입대 전 준비"
author_profile: false
classes: wide
sitemap: false
---

<div class="baf-wrap" markdown="0">
  <p class="baf-title">입대 전 금융 준비 정리</p>
  <div id="baf-content"></div>
</div>

<script>
(function () {
  var html = sessionStorage.getItem("baf-content");
  if (!html) {
    window.location.replace("/before-airforce/");
    return;
  }
  document.getElementById("baf-content").innerHTML = html;

  document.querySelectorAll(".baf-checklist input[type=checkbox]").forEach(function (box) {
    var storeKey = "baf-check-" + box.dataset.key;
    box.checked = localStorage.getItem(storeKey) === "1";
    box.addEventListener("change", function () {
      localStorage.setItem(storeKey, box.checked ? "1" : "0");
    });
  });
})();
</script>
