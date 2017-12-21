---
title: AWS SNS Using Rails
date: 2017-11-09
tags: AWS SNS
---
---

#How to set up SNS on AWS so you can track bounce, complaint and delivery from Rails.

Sending emails using AWS smtp from Rails is easy. Initially new users are placed in the Amazon SES sandbox. Here, the user can send mails within the verified email addresses and domains. Also the maximum limit of sending messages is 200/day. To improve this restriction, user have to open case in support center.

Once the user moved out from sandbox mode, user can able to mail anywhere through the verified domain/email addresses. AWS provides a concept called SNS (simple notification service) where the user can handle bounce, complaint and delivery rate. This notification source can be of email, http/https JSON type.

###Add bounce/complaint/delivery handler to Rails App

#### 1. DB schema
```ruby?line_numbers=false
class CreateTrackings < ActiveRecord::Migration
  def change
    create_table :trackings do |t|
      t.string :source
      t.string :destination
      t.string :message_id
      t.string :notify_type
      t.string :bounce_type
      t.string :bounce_subtype
      t.string :bounce_action
      t.string :bounce_status
      t.text :bounce_diagnostic_code
      t.boolean :is_delivered
      t.boolean :is_spam
      t.text :delivered_response
      t.date :mail_timestamp
      t.timestamps
    end
  end
end
  ```

#### 2. Model
```ruby?line_numbers=false
  class Tracking < ActiveRecord::Base
    validates_presence_of :destination
  end
```
#### 3. Gemfile
```ruby?line_numbers=false
  gem "aws_sns_subscription"
```
#### 4. Controller
```ruby?line_numbers=false
  require 'json'
  class AwsController < ApplicationController
  skip_before_action :verify_authenticity_token
  before_filter :respond_to_aws_sns_subscription_confirmations
  before_action :data, :message

  def bounce
    begin
    # raise exception if the request is not of bounce type
    raise unless message['notificationType'] == "Bounce"
    mail = message['mail']
    bounce = message['bounce']
    recipients = bounce['bouncedRecipients']
    # process each delivery recipients and update in the table
    recipients.each do |recp|
    @track = validation[:domain].trackings.where(source: mail['source'], message_id: mail['messageId']).first
      if @track.blank?
        validation[:domain].trackings.create(mail_timestamp: mail['timestamp'].to_date, source: mail['source'], message_id: mail['messageId'], destination: recp['emailAddress'], notify_type: message['notificationType'], bounce_type: bounce['bounceType'], bounce_subtype: bounce['bounceSubType'], bounce_action: recp['action'], bounce_status: recp['status'], bounce_diagnostic_code: recp['diagnosticCode'])
      else
        @track.update(mail_timestamp: mail['timestamp'].to_date, destination: recp['emailAddress'], notify_type: message['notificationType'], bounce_type: bounce['bounceType'], bounce_subtype: bounce['bounceSubType'], bounce_action: recp['action'], bounce_status: recp['status'], bounce_diagnostic_code: recp['diagnosticCode'])
      end
    end
      render json: {}
    rescue Exception => e
      render json: {}
    end
  end

  def complaint
    begin
      # raise exception if the request is not of complaint type
      raise unless message['notificationType'] == "Complaint"
      mail = message['mail']
      complaint = message['complaint']
      recipients = complaint['complainedRecipients']
      # process each delivery recipients and update in the table
      recipients.each do |recp|
        @track = validation[:domain].trackings.where(source: mail['source'], message_id: mail['messageId']).first
        if @track.blank?
          validation[:domain].trackings.create(mail_timestamp: mail['timestamp'].to_date, source: mail['source'], message_id: mail['messageId'], destination: recp['emailAddress'], notify_type: message['notificationType'], is_spam: true)
        else
          @track.update(mail_timestamp: mail['timestamp'].to_date, destination: recp['emailAddress'], notify_type: message['notificationType'], is_spam: true)
        end
      end
      render json: {}
    rescue Exception => e
      render json: {}
    end
  end

  def delivered
    begin
      # raise exception if the request is not of delivery type
      raise unless message['notificationType'] == "Delivery"

      mail = message['mail']
      delivery = message['delivery']
      recipients = delivery['recipients']
      # process each delivery recipients and update in the table
      recipients.each do |recp|
        @track = validation[:domain].trackings.where(source: mail['source'], message_id: mail['messageId']).first
        if @track.blank?
          validation[:domain].trackings.create(mail_timestamp: mail['timestamp'].to_date, source: mail['source'], message_id: mail['messageId'], destination: recp, notify_type: message['notificationType'], is_delivered: true, delivered_response: delivery['smtpResponse'])
        else
          @track.update(mail_timestamp: mail['timestamp'].to_date, destination: recp, notify_type: message['notificationType'], is_delivered: true, delivered_response: delivery['smtpResponse'])
        end
      end
      render json: {}
    rescue Exception => e
      render json: {}
    end
  end

  protected

    def data
      JSON.parse(request.raw_post)
    end

    def message
      message ||= JSON.parse JSON.parse(request.raw_post)['Message']
    end
  end
```

#### 5. Routes

```ruby?line_numbers=false
  post    "/aws/handler/bounce"            =>  "aws#bounce"
  post    "/aws/handler/complaint"         =>  "aws#complaint"
  post    "/aws/handler/delivered"         =>  "aws#delivered"
```

Create http/https endpoint to the required topic with the provided url's (routes.rb).

Migrate the database, install the gem and start handling the tracks.
