---
layout: post
title: Providing personal push reporting via Tableau
tags:
- Tableau
---

Tableau is one of my favorite Business Intelligence platforms: it achieves an excellent balance of flexibility and control and allows users to explore the data and create new insights. However, the pricing can be prohibitively expensive for small companies ($1k per user up front, $200 per year afterwards) and not all members of the organization require the full interactor experience: some people just want to see what happened yesterday, last week, or last month. As a result, many companies start with just a limited use PoC and miss a great opportunity to get the organization used to looking at the numbers. At TravelBird we've built a Python script to allow everyone in the company (even those without Tableau licenses) to receive a personalized report every day.

###The key: tabcmd and URL filters

One of the most powerful Tableau tools available to a BI Manager who wants to share data is tabcmd, which allows incredible functionality to generate images, export PDFs and data, and manage a number of other tasks on the server. For example, if we imagine our server is _https://reporting.MyCompany.com_, the workbook is _Daily Sales Report_, and the default view is _Yesterday Sales Summary_, then to generate a PNG summary, PDF report, and the underlying data we could execute:

```
tabcmd login -s https://reporting.MyCompany.com -u <admin_username> -p <admin_password>
tabcmd export "Daily Sales Report/Yesterday Sales Summary" --pdf -f "C:\output\Sales.pdf"
tabcmd get "Daily Sales Report/Yesterday Sales Summary" --png -f "C:\output\summary.png"
tabcmd export "Daily Sales Report/Yesterday Sales Summary" --csv -f "C:\output'sales.csv"
tabcmd logout
```

As imaginable, this could all be compiled into an email using a tool like [Blat](http://www.blat.net/) and distributed to a group. However, the trick to personal emails are Tableau's URL filters, which can be used both on the web and with tabcmd. For example, using _https://reporting.mycompany.com/#/DailySalesReport/YesterdaySalesSummary_ will show the organization's total performance, but adding **?SalesPerson=Joe Smith** to the end will filter the data to only Joe. The real power to this trick is that **the filter does not need to be visible on the dashboard, only present in the workbook filters**. Provided the reports are built with the desired filter in the data set, almost any report can be automatically personalized on distribution.

###Distributing personal reports

To make this work, three things are required: the base report to distribute, a table containing the filter clause and recipient email, and an email provider who plays well with Python (we use [SendGrid](https://sendgrid.com/) as they have a convenient library). In our job we've added additional controls to provide multiple scheduling options, choose on a recipient basis what attachments are included, and track the last successful distribution, but the base job is relatively straightforward (presume the reference table contains salesperson, region, and email):

```python
import sendgrid
import pyodbc
import os
import datetime

sendgrid_user = 'my_user'
sendgrid_pass = 'my_pass'

tmp_folder = 'C:\\temp\\'
tableau_folder = 'C:\\Program Files\\Tableau\\Tableau Server\
    \\9.0\\bin\\tabcmd.exe'
tab_user = 'my_admin'
tab_pass = 'my_admin_pass'


def return_recipients():
    odbc_cxn = pyodbc.connect('DSN=my_db')
    cxn = odbc_cxn.cursor()
    cxn.execute("select salesperson_name, region, salesperson_email from ref_table")
    results = cxn.fetchall()
    cxn.close()
    odbc_cxn.close()


def generate_files(baseReport, fileName, filteredName):
    attachments = []
    os.command(tableau_folder +
        ' login -s https://reporting.travelbird.com -u %s -p %s') % \
        (tab_user, tab_pass)
    os.command(tableau_folder + ' export %s --png -f "%s%s.png"') % \
        (filteredName, tmp_folder, fileName)
    attachments.append('%s.png' % fileName)
        os.command(tableau_folder + ' export %s --pdf -f "%s%s.pdf"') % \
            (filteredName, tmp_folder, fileName)
        attachments.append('%s.pdf' % fileName)
        os.command(tableau_folder + ' export %s --csv -f "%s%s.csv"') % \
            (filteredName, tmp_folder, fileName)
        attachments.append('%s.csv' % fileName)
    os.command(tableau_folder + ' logout')
    return attachments


def send_email(user, toAddress, subject, attachments):
    sg = sendgrid.SendGridClient(sendgrid_user, sendgrid_pass)
    message = sendgrid.Mail()
    message.set_from(from_email)
    message.set_subject(subject)
    body = 'Good morning %s,\n' % user
    body += 'Attached is the %s report for %s.\n\n' % \
        (subject, time.strftime('%a, %d %b %Y'))
    body += 'For questions, please contact bi@MyCompany.com'
    message.set_text(body)
    message.add_to(toAddress)
    for attachment in attachments:
        message.add_attachment(attachment, tmp_folder + attachment)
    sg.send(message)


def main():
    baseReport = 'Daily Sales Report/Yesterday Sales Summary'
    fileName = baseReport.split('/')[0]
    email_list = return_recipients()
    for recipient in email_list:
        filteredName = baseReport + "?Region=" + recipient.region
        attachments = generate_files(baseReport, fileName, filteredName)
        send_email(recipient.salesperson_name, recipient.salesperson_email, fileName, attachments)
        for file in attachments:
            os.remove(file)


if __name__ == '__main__':
    main()
```

And that's it! With this small piece of code, every member in the organization can start the day with their personalized report and have the full data if they need to do further analysis. As opposed to the Tableau email platform, this script based approach can offer a huge amount of flexibility:
* Central BI management of distributions and distributed content
* Database checks can be added for given reports (ex. check that sales data is complete at 7 AM before sending out the PDF)
* Multiple reports or PNGs could be consolidated into one email
* The entire message body can be tuned/customized to meet business needs. For example, populating the summary statistics into the subject for better experience


What might it look like? With the Superstore sample data:

![personalized report]({{site.url}}/assets/personal_report_example.png)

