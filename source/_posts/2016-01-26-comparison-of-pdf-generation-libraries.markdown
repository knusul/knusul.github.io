---
layout: post
title: "Comparison of PDF generation libraries"
date: 2015-06-26 15:48:34 +0000
comments: true
categories: 
---

Recently I investigated the possibilities of PDF generation from doc(x)
documents. It came up that its not that trivial.
The goal is to have separate application that will have api to perform conversions.

# Example Usage:

Post: docx2pdf.tmh.io/templates/create?filename=name.docx # status: ok
Get: doc2pdf.tmh.io/templates/1/pdf?placeholder1=value1&placeholder2=value2  #document.pdf
