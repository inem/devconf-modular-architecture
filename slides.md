# From rails-way to modular architecture

## Ivan Nemytchenko, independent consultant

### DevConf 2014

### @inem, @inemation


---
![](devconf/graduate.png)
![](devconf/guru.png)

```
Icon made by <a href="http://www.freepik.com" alt="Freepik.com">Freepik</a>
from <a href="http://www.flaticon.com/free-icon/graduate-cap_46045">flaticon.com</a>

Icon made by Icons8 from <a href="http://www.flaticon.com">flaticon.com</a>
```

---

![fit](devconf/rails-community.jpg)

---

![filtered](devconf/rails-community.jpg)

* [bit.ly/rails-community](http://bit.ly/rails-community)

---

![fit](devconf/clean_a.jpg)

---

# [fit] 🚩🚀---→----→----→----→----📍

---

# Rails-way

---

# Модульная архитектура - это что?

* изменения в коде делать легко
* код легко переиспользовать
* код легко тестировать

---

# Чем плох rails-way?

---

# Single Responsibility Principle

##  - Не, не слышали

---

# Skinny controllers, fat models.

# ORLY?

---

# Conventions over configuration

# DB ⇆ Forms

---
# Проект

* Монолитное приложение на Grails
* База данных на 70 таблиц
* Мы упоролись

---
# Решение заказчика

* Фронтенд на AngularJS
* Бэкэнд на рельсе для отдачи API

---
# Ок, rails, значит rails

## Точнее, rails-api

---
# Модели

  ```ruby
    class ImageSettings < ActiveRecord::Base
    end

    class Profile < ActiveRecord::Base
      self.table_name = 'profile'
      belongs_to :image, foreign_key: :picture_id
    end
  ```

---

![fit](db.jpg)

---
# Модели

  ```ruby
    class Image < ActiveRecord::Base
      self.table_name = 'image'
      has_and_belongs_to_many :image_variants,
          join_table: "image_image",
          class_name: "Image",
          association_foreign_key: :image_images_id

      belongs_to :settings,
          foreign_key: :settings_id,
          class_name: 'ImageSettings'

      belongs_to :asset
    end
  ```

---
#RABL - [github.com/nesquena/rabl](https://github.com/nesquena/rabl)

  ```ruby
    collection @object
    attribute :id, :deleted, :username, :age

    node :gender do |object|
      object.gender.to_s
    end

    node :thumbnail_image_url do |obj|
      obj.thumbnail_image.asset.url
    end

    node :standard_image_url do |obj|
      obj.standard_image.asset.url
    end
  ```

---
# [fit] 🚩-🚀--→----→----→----→----📍

---

  ```ruby
    def redeem
      unless bonuscode = Bonuscode.find_by_hash(params[:code])
        render json: {error: 'Bonuscode not found'}, status: 404 and return
      end
      if bonuscode.used?
        render json: {error: 'Bonuscode is already used'}, status: 404 and return
      end
      unless recipient = User.find_by_id(params[:receptor_id])
        render json: {error: 'Recipient not found'}, status: 404 and return
      end
      unless ['regular', 'paying'].include?(recipient.type)
        render json: {error: 'Incorrect user type'}, status: 404 and return
      end

      ActiveRecord::Base.transaction do
        amount = bonuscode.mark_as_used!(params[:receptor_id])
        recipient.increase_balance!(amount)

        if recipient.save && bonuscode.save
          render json: {balance: recipient.balance}, status: 200 and return
        else
          render json: {error: 'Error during transaction'}, status: 500 and return
        end
      end
    end
  ```

---

  ![fit](devconf/controller-business-logic.jpg)

---

  ```ruby
    def redeem
      begin
        recipient_balance = ??????????
      rescue BonuscodeNotFound, BonuscodeIsAlreadyUsed, RecipientNotFound => ex
        render json: {error: ex.message}, status: 404 and return
      rescue IncorrectRecipientType => ex
        render json: {error: ex.message}, status: 403 and return
      rescue TransactionError => ex
        render json: {error: ex.message}, status: 500 and return
      end

      render json: {balance: recipient_balance}
    end
  ```

---

#  [fit]  bonuscode.redeem(user)
#             или
#  [fit]  user.redeem_bonuscode(code)
#              ?

---

# Service/Use case:

  ```ruby
    def redeem
      use_case = RedeemBonuscode.new

      begin
        recipient_balance = use_case.run!(params[:code], params[:receptor_id])
      rescue BonuscodeNotFound, BonuscodeIsAlreadyUsed, RecipientNotFound => ex
        render json: {error: ex.message}, status: 404 and return
      rescue IncorrectRecipientType => ex
        render json: {error: ex.message}, status: 403 and return
      rescue TransactionError => ex
        render json: {error: ex.message}, status: 500 and return
      end

      render json: {balance: recipient_balance}
    end
  ```

---

  ```ruby
    class RedeemBonuscode
      def run!(hashcode, recipient_id)
        raise BonuscodeNotFound.new unless bonuscode = find_bonuscode(hashcode)
        raise RecipientNotFound.new unless recipient = find_recipient(recipient_id)
        raise BonuscodeIsAlreadyUsed.new if bonuscode.used?
        raise IncorrectRecipientType.new unless correct_user_type?(recipient.type)

        ActiveRecord::Base.transaction do
          amount = bonuscode.redeem!(recipient_id)
          recipient.increase_balance!(amount)
          recipient.save! && bonuscode.save!
        end
        recipient.balance
      end

      private
      ...
    end

  ```
---
# [fit] 🚩---→-🚀---→----→----→----📍

---
![left](devconf/incoming2.jpg)

# Incoming!!!
* В таблице "users" по факту хранятся разные типы пользователей и для них нужны разные правила валидации
* Это как минимум

---
## Themis - modular and switchable validations for ActiveRecord models

```ruby
  class User < ActiveRecord::Base
    has_validation :admin, AdminValidation
    has_validation :enduser, EndUserValidation
    ...

  user.use_validation(:enduser)
  user.valid? # => true

```

* [github.com/TMXCredit/themis](http://github.com/TMXCredit/themis)

---

  ```ruby
    module EndUserValidation
      extend Themis::Validation

      validates_presence_of :username, :password, :email
      validates :password, length: { minimum: 6 }
      validates :email, email: true
    end
  ```

---

# [fit] 🚩---→---🚀-→----→----→----📍

# Form objects (Inputs)
* Plain old ruby objects*

```ruby
  class BonuscodeRedeemInput < Input
    attribute :hash_code, String
    attribute :receptor_id, Integer

    validates_presence_of :hash_code, :receptor_id
    validates_numericality_of :receptor_id
  end
```

---

# Form objects (Inputs)

  ```ruby
    class Input
      include Virtus.model
      include ActiveModel::Validations

      class ValidationError < StandardError; end

      def validate!
        raise ValidationError.new, errors unless valid?
      end
    end

  ```

---

# [fit] 🚩---→----→---🚀-→----→----📍

---

# Incoming!
## Заказчику нужен небольшой сервис сбоку


  ```bash
  mkdir sinatra-app
  cd sinatra-app
  git init .

  ```

---

  ```ruby
    class SenderApp < Sinatra::Base
      get '/sms/create/:message/:number' do
        input = MessageInput.new(params)
        sender = TwilioMessageSender.new('xxxxxx', 'xxxxxx', '+15005550006')
        use_case = SendSms.new(input, sender)

        begin
          use_case.run!
        rescue ValidationError => ex
          status 403 and return ex.message
        rescue SendingError => ex
          status 500 and return ex.message
        end
        status 201
      end
    end
  ```

---
# Sinatra

* встраивается в рельсовые роуты или запускается в связке с другими rack-приложениями
* рельсовые роуты оказались ненужной абстракцией
* отчего бы не применить такой подход для всего приложения?!

---
# [fit] 🚩---→----→----→--🚀--→----📍

---
# А как же разделение бизнес-логики и хранения данных?
* Очень хочется но непонятно как
* Репозитории?


---

# Entities

---
# Entities - зачем?

```
statuses        profiles
--------        --------
id              id
name            status_id

```


```ruby
profile.status.name

```

---

# Entities

* Plain old ruby objects*

```ruby
  class User < Entity
    attribute :id, Integer
    attribute :username, String
    attribute :max_profiles, Integer, default: 3

    attribute :profiles, Array[Profile]
    attribute :roles, Array
  end

```

---
# Entities

* Plain old ruby objects*

```ruby
  class Entity
    include Virtus.model

    def new_record?
      !id
    end
  end
```
---

# Repositories

---

# Repositories - чем занимаются?

* найди мне объект с таким-то id
* дай мне все объекты по такому-то условию
* сохрани вот эти данные в виде объекта

---

# AR-flavored repositories

* под капотом все те же ActiveRecord модели
* репозиторий отдает инстансы Entities

---
Первая попытка:

# AR-flavored repositories
# NEVER AGAIN!

---
# Sequel FTW!
## [sequel.jeremyevans.net](http://sequel.jeremyevans.net/)

* DSL для построения SQL-запросов
* реализация паттерна ActiveRecord

---
# Repositories

```ruby
  def find(id)
    dataset = table.select(:id, :username, :enabled,
                :date_created, :last_updated).where(id: id)
    user = User.new(dataset.first)
    user.roles = get_roles(id)
    user
  end
```

---
# Repositories

```ruby
def find(id)
  to_add = [:gender__name___gender, :profile_status__name___status,
            :picture_id___image_id]

  dataset = table.join(:profile_status, id: :profile_status_id)
                 .join(:gender, id: :profile__gender_id)
                 .select_all(:profile).select_append(*to_add)
                 .where(profile__id: id)

  all = dataset.all.map do |record|
    Profile.new(record)
  end
  all.size > 1 ? all : all.first
end
```
---
# Repositories

```ruby
def persist(profile)
  status_id = DB[:online_profile_status].select(:id).where(name: profile.status).get(:id)

  gender_id = DB[:gender].select(:id).where(name: profile.gender).get(:id)

  hash = { username: profile.username,
           profile_status_id: status_id,
           gender_id: gender_id,
           picture_id: profile.image_id }

  if profile.new_record?
    dates = { date_created: Time.now.utc, last_updated: Time.now.utc }
    profile.id = table.insert(hash.merge! dates)
  else
    table.where(id: profile.id).update(hash)
  end
end
```
---
# Repositories

```ruby
class ImageRepo
  class ImageData < Sequel::Model
    set_dataset DB[:image].join(:asset, asset__id: :image__asset_id)
    many_to_many :images,
                 join_table: :image_image,
                 left_key: :image_id,
                 right_key: :image_images_id,
                 class: self
  end
...
```
---

# Benefits?

---

# [fit] 🚩---→----→----→----→---🚀-📍

---
# Presenters

---

```ruby
class ProfilePresenter
  def initialize(p, viewer)
    ...

  def wrap!
    hash = { id:        p.id,       deleted:   p.deleted,
             username:  p.username, is_online: p.is_online }
    hash[:user_id] = p.user_id       if viewer.admin?
    hash[:image]   = p.image.to_hash if add_image?
    hash
  end

  def self.wrap!(profiles, viewer)
    profiles.map do |profile|
      new(profile, viewer).wrap!
    end
  end
  ...
end
```

---
# Presenters

```ruby
  post '/profile' do
    begin
      use_case = CreateProfile.new(current_user)
      profile = use_case.run!(params)
    rescue Input::ValidationError => ex
      halt 403
    end

    wrapped_profiles = ProfilePresenter.wrap!([profile], current_user)
    json(data: wrapped_profiles)
  end
```
---

# [fit] 🚩---→----→----→----→----🚀📍

---

# [Hexagonal/Clean Architecture](http://blog.8thlight.com/uncle-bob/2012/08/13/the-clean-architecture.html)

---
![fit](devconf/clean_a.jpg)

---

![fit](devconf/circles.jpg)

---

# From rails-way to modular architecture

## Ivan Nemytchenko, independent consultant

### DevConf 2014

### @inem, @inemation
