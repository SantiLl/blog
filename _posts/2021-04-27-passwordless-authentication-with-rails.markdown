---
layout: post
title:  "Autenticación Passwordless con Rails"
date:   2021-04-27 23:00:00 -0300
permalink: autenticacion-passwordless-con-rails
---
# ¿Qué es una autenticación passwordless?
Una autenticación passwordless es una manera de verificar un usuario en nuestra app sin necesidad de que requieran un nombre de usuario y contraseña. En cambio, vamos a generar un token de seguridad para autenticar al usuario.

Estos son algunos de los beneficios que otorgan tener una autenticación passwordless: 
- una UX mejorada, los usuarios no tienen la necesidad de recordar sus contraseñas y generar una nueva en el caso de que se olviden la antigua.
- aumento en seguridad, se reduce la posibilidad de el reutilización de la misma contraseña en el caso de que surja un hackeo/leak en algunas de las plataformas en las que se haya registrado. También imposibilita la probabilidad de phishing.
- relativamente fácil de crear, se simplifica el proceso de login.

# Escenario de uso:
Los casos de uso con esta funcionalidad son infinitos. En este ejemplo en particular lo voy a asemejar a un trabajo que tuve en un cliente, en donde hice uso de esta funcionalidad, pero con una versión simplificada porque sino el articulo se me va de las manos. No vamos a hacer uso de gemas como `devise` o `sorcery`, aunque se podría realizar de igual manera con ellas.

# Setup:
Vamos a crear la app: `rails new passwordless-auth-app -T --skip-turbokinks --database=postgresql`

## 1. Instalación de gemas:
*NOTA*: las gemas que utilizo no son necesarias para implementar esta funcionalidad.
````ruby
# ./Gemfile.rb
# formulario para ingreso del email
gem 'simple_form', '~> 5.1'

group :development, :test do
  # para chequear el envio de mails en entorno de desarrollo
  gem 'letter_opener', '~> 1.7' 
end
````
`bundle install` debería instalar todas las gemas. Luego de esto vamos a setear la configuración para `letter_opener`.
````ruby
# ./config/environments/development.rb
Rails.application.configure do
  config.action_mailer.default_url_options = { host: "http://localhost:3000" }
  config.action_mailer.delivery_method = :letter_opener
  (...)
end
````

## 2. Creación del esqueleto de la app:

### Migraciones de modelos y controllers:
````ruby
# Modelos
rails g model User email:string login_token:string login_token_expires_at:datetime &&
rake db:create && 
rake db:migrate

# Controllers
rails g controller sessions index
rails g controller users new create
````

### Modificación de rutas:
````ruby 
# ./config/routes.rb
Rails.application.routes.draw do
  resources :sessions, only: [:index] do
    get :undefined_login_token, on: :collection
  end

  resources :users, only: [:new, :create] do
    post :update_new_access_token, on: :collection
    get :generate_new_access_token, on: :collection
    get :email_sent, on: :member
  end

  root to: 'users#new'
end
````

### Modelo, callbacks, y, metodos de instancia:
Ahora viene la parte mas divertida, en donde nos vamos a ensuciar con un poco de lógica. Vamos a crear la función `generate_login_token!` en donde actualizamos el login_token con un valor alfanumérico al azar y la fecha de expiración de ese token. A su vez vamos a setear un [callback](https://guides.rubyonrails.org/active_record_callbacks.html){:target="_blank"} para que esta función se ejecute cada vez que creamos un nuevo record.
Por otro lado en `login_token_url` ya dejamos seteado el path para que el usuario pueda acceder a nuestra app (recuerden cambiar esta función si van a subir su app a producción ya que va a romper porque `localhost:3000` va a ser una url inexistente).
````ruby
# ./models/user.rb
class User < ApplicationRecord

  # Callbacks
  before_create :generate_login_token!

  # Instance Methods
  def generate_login_token!
    self.login_token = SecureRandom.urlsafe_base64
    self.login_token_expires_at = 10.days.from_now
  end

  def login_token_valid?
    login_token_expires_at > Time.now
  end

  def login_token_url
    # NOTA: Cambiar la url cuando la app este en produccion, se puede hacer por medio de la gema dotenv o rails credentials.
    "http://localhost:3000/sessions?login_token=#{login_token}"
  end
end
````

### Controller:
El controller no tiene mucha logica, paso a explicar cual sería el objetivo de cada path:

- *new*: va a ser donde el user ingrese su email para que le envíen el `login_token`.
- *create*: Es un create básico.
- *generate_new_access_token*: En el caso de que el usuario se olvide/vence el `login_token`, puede entrar a esta vista donde se le pediría que introduzca el email de su usuario ya registrado.
- *update_new_access_token*: Va a ser el path al que se va redireccionar desde la vista de `generate_new_access_token` para generar un nuevo `login_token` con el refresh de `login_token_expires_at`.


````ruby
# ./app/controllers/users_controller.rb
class UsersController < ApplicationController
  before_action :set_user, only: %i[:email_sent]
  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)
    redirect_to email_sent_user_path(@user) if @user.save
  end

  def email_sent
  end

  def generate_new_access_token
  end

  def update_new_access_token
    @user = User.find_by(user_params)
    if @user.present?
      @user.generate_login_token!
      @user.save
      redirect_to generate_new_access_token_users_path, notice: "Login token was successfully updated, check your email."
    else
      redirect_to undefined_login_token_sessions_path
    end
  end

  private

  def set_user
    @user = User.find(params[:user_id])
  end

  def user_params
    params.require(:user).permit(:email)
  end
end
````

### Vistas:
Las vistas las simplifique lo mas posible, con ver el contenido se entiende el objetivo que buscan en cuanto a la UX.
````ruby
# ./app/views/users/new.html.erb
<h1>Bienvenido a passwordless-auth-app!</h1>
<h2>Introudce to email abajo en el formulario para registrarte.</h2>
<%= simple_form_for(@user) do |f| %>
  <%= f.input :email %>
  <%= f.button :submit %>
<% end %>
````

````ruby
# ./app/views/sessions/email_sent.html.erb
<h1>Gracias por registrarte! Se te ha enviado un email a <%= @user.email %> con el accesso a la plataforma</h1>
````

````ruby
# ./app/views/users/generate_new_access_token.html.erb
<h1>Genera un nuevo login token en el caso de que se te haya expirado o no lo recuerdes</h1>
<%= simple_form_for(:user, url: update_new_access_token_users_path, method: :post) do |f| %>
  <%= f.input :email %>
  <%= f.button :submit %>
<% end %>
````

````ruby
# ./app/views/sessions/undefined_login_token.html.erb
<h1>No has podido acceder a tu cuenta?</h1>
<h2>Si se expiro tu login_token, genera uno nuevo <%= link_to 'aqui', generate_new_access_token_users_path %></h2>
<h2>No tienes cuenta? <%= link_to 'Registrate', new_user_path %> </h2>
````

````ruby
# ./app/views/sessions/index.html.erb
<h1>Bienvenido <%= @user.email %></h1>
<p>Aqui puedes administrar y modificar todos tus datos.</p>
````


## 3. Agregado de notificaciones para enviar los emails:
La frutillita del postre, configurar el proceso de ejecución del mail en el que se envíe el `login_token`. 

###Iniciamos creando una el mailer:
````ruby
rails generate mailer LoginMailer
````
Luego agregamos la función `send_email` dentro de la clase `LoginMailer`:
````ruby
# ./app/mailers/login_mailer.rb
class LoginMailer < ApplicationMailer
  def send_email(user, url)
    @user = user
    @url  = url

    mail to: @user.email, subject: 'Ingresar a paswordless-auth-app'
  end
end
````

Y finalmente agregamos la vista de este mailer:
````ruby
# ./app/views/login_mailer/send_email.html.erb
<!DOCTYPE html>
<html>
  <head>
    <meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
  </head>
  <body>
    <h1>Bienvenido <%= @user.email %>,</h1>
    <a href="<%= @url %>">Haz click para acceder a la app.</a>
</html>
````

### Callback para la ejecución del mail:
Para tener el proceso completo sería generar una ejecución automática de este mail una vez que se crea el usuario/renueva el `login_token`.
````ruby
# ./app/models/user.rb
class User < ApplicationRecord
  # Callback
  after_save :send_new_login_token_notification, if: Proc.new { saved_change_to_attribute?(:login_token) }
  
  # Instance Method
  def send_new_login_token_notification
    LoginMailer.send_email(self, login_token_url).deliver_now
  end
end
````

## 4. Autenticación de la sesión:
Por ultimo paso queda validar que el `login_token` sea el correcto dentro de `SessionsController`:

````ruby
class SessionsController < ApplicationController
  def index
    @user = User.find_by(login_token: login_token_params)

    unless @user.present? && @user.login_token_valid?
      redirect_to undefined_login_token_sessions_path
    end
  end

  def undefined_login_token
  end

  private

  def login_token_params
    params.require(:login_token)
  end
end
````

## 5. Et Voaià!
<img src="https://res.cloudinary.com/dd28ghazj/image/upload/v1619574813/santiagollapur.com/autenticacion-passwordless-con-rails/passwordless-auth_ci3xqn.gif">
Ya tenemos funcionando la autenticación de usuarios sin necesidad de tener una contraseña 🎉. 

El objetivo de este post es dar una idea básica de una de las muchas formas en las que se puede crear este tipo de autenticaciones, hay mucho mas en términos de seguridad y autorización para trabajar. 

Les comparto los 2 principales posts en los que me sirvieron para realizar este trabajo:

- [Magic Links with Ruby On Rails and Devise](https://dev.to/matiascarpintini/magic-links-with-ruby-on-rails-and-devise-4e3o){:target="_blank"}

- [Implement a passwordless authentication with Sorcery](https://fullstackheroes.com/rails/sorcery-passwordless-authentication/){:target="_blank"}
