+++
title = "Contact"
description = "Where to finde me"
date = "2023-03-02"
author = "Stephan Schröder"
+++

<form class="mb-5" id="contact-form">
  <div class="form-group">
    <label for="senderEmailId" style="width: 400px">your email:</label>
    <br>
    <input type="email" class="form-control" id="senderEmailId" placeholder="name@example.com" name="senderEmail">
    <br>
    <small id="emailHelp" class="form-text text-muted">optional, only necessary if you want an answer.</small>
  </div>

  <div class="form-group">
    <label for="subjectId" data-width="50%">subject:</label>
    <br>
    <input type="text" class="form-control" id="subjectId" placeholder="subject" name="subject">
  </div>

  <div class="form-group">
    <label for="messageId">message:</label>
    <br>
    <textarea class="form-control" id="messageId" rows="8" name="message"></textarea>
  </div>

  <input type="hidden" name="receiverName" value="void2unitBlog">

  <span id="contactAlertParent"></span>

  <button type="submit" id="contactButtonId" class="btn btn-primary">Send</button>
</form>

<script type="text/javascript" src="/js/jquery-3.7.1.min.js"></script>
<script type="text/javascript" src="/js/contact-form-handler.js"></script>
