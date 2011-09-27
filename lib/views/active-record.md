#### {% title "ActiveRecord w przykładach" %}

Lista przykładów:

1.  Why Associations?
1.  Badamy powiązanie wiele do wielu na konsoli


*TODO*:

6.  Dopisywanie rekordów do bazy :action => 'authors'
7.  Pobieranie rekordów za pomocą <i>find</i> :action => 'find'
8.  Przeglądanie pobranych rekordów :action => 'iterate'
9.  Efektywne pobieranie rekordów z kliku tabel :action => 'retrieving_data'
10. Uaktualnianie danych :action => 'updating_data'
11. Wymuszanie integralności danych :action => 'validation'
13. Zapobieganie „race conditions” :action => 'transactions'
14. Ręczne sortowanie z użyciem acts_as_list :action => 'sorting'
7.  Sortowanie za pomocą drag &amp; drop :action => 'drag_and_drop'
21. Powiązania polimorficzne :action => 'polymorphic_associations'
XX. Coś prostego: single-table inheritance :action => 'single-table-inheritance'

Zaczniemy od sprawdzenia jakie mamy zainstalowane w systemie
Rubies i zestawy gemów:

    rvm list
    rvm list gemsets

Następnie, na potrzeby przykładów z tego wykładu, utworzymy zestaw
gemów o nazwie *ar*:

    rvm use --create ruby-1.9.2-p290@ar
    gem install bundler rails
    rvm info
    rvm current

Co daje takie podejście?


## Why Associations?

Przykład z rozdziału 1
[A Guide to Active Record Associations](http://guides.rubyonrails.org/association_basics.html).
Zobacz też [ActiveRecord::ConnectionAdapters::TableDefinition](http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/TableDefinition.html).

Generujemy przykładową aplikację:

    rvm use ruby-1.9.2-p290@ar
    rails new why_associations --skip-bundle
    cd why_associations
    bundle install --path=.bundle/gems

Przy okazji ułatwimy sobie śledzenie kolejności
migracji dopisując do pliku *config/application.rb*:

    :::ruby
    config.active_record.timestamped_migrations = false

Skorzystamy z generatora do wygenerowania kodu dla dwóch modeli:

* *Customer* (klient) – atrybuty: *name*, …
* *Order* (zamówienie) – atrybuty: *order_date*, *order_number* …

Każdy klient może mieć wiele zamówień. Każde zamówienie
należy do jednego kilenta. Dlatego między kilentami i zamówieniami
mamy powiązanie jeden do wielu. Oznacza to, że powinniśmy dodać
jeszcze jeden atrybut (*foreign key*) do zamówień:

* *Order* (zamówienie) – atrybuty: **customer_id**, order_date, …

Generujemy *boilerplate code*::

    rails generate model Customer name:string
    rails generate model Order order_date:datetime order_number:string customer:references

Migrujemy i przechodzimy na konsolę:

    rake db:migrate
    rails console

gdzie dodajemy klienta i dwa jego zamówienia:

    :::ruby
    Customer.create :name => 'wlodek'
    @customer = Customer.first  # jakiś klient
    Order.create :order_date => Time.now, :order_number => '20111003/1', :customer_id => @customer.id
    Order.create :order_date => Time.now, :order_number => '20111003/2', :customer_id => @customer.id
    Order.all

A tak usuwamy z bazy klienta i wszystkie jego zamówienia:

    :::ruby
    @customer = Customer.first
    @orders = Order.where :customer_id => @customer.id
    @orders.each { |order| order.destroy }
    @customer.destroy


### Dodajemy powiązania między modelami

Po dopisaniu kod usalającego powiązania między tymi modelami:

    :::ruby
    class Customer < ActiveRecord::Base
      has_many :orders, :dependent => :destroy
    end
    class Order < ActiveRecord::Base
      belongs_to :customer
    end

tworzenie nowych zamówień dla danego klienta jest łatwiejsze:

    :::ruby
    Customer.create(:name => 'rysiek')
    @customer = Customer.where(:name=> 'rysiek').first
    @order = @customer.orders.create :order_date => Time.now, :order_number => '20111003/3'
    @order = @customer.orders.create :order_date => Time.now, :order_number => '20111003/4'

Usunięcie kilenta wraz z wszystkimi jego zamówieniami jest też proste:

    :::ruby
    @customer = Customer.first  # jakiś klient
    @customer.destroy

Tak jest prościej.


## Badamy powiązanie wiele do wielu na konsoli

Między zasobami – *assets* i opisującymi je cechami – *tags*.

Przykład pokazujący o co nam chodzi:

* *Asset*: Cypress. A photo of a tree. (dwa atrybuty)
* *Tag*: tree, organic

Między zasobami i opisującymi je cechami mamy powiązanie wiele do
wielu.

Dodatkowo, każdy zasób przypisujemy do jednego z kilku rodzajów zasobów –
*AssetType*.  Przykład:

* *Asset*: Cypress. A photo of a tree.
* *AssetType*: Photo

Między zasobem a rodzajem zasobu mamy powiązanie wiele do jednego.

Tak jak poprzednio skorzystamy z generatora do wygenerowania
**boilerplate code**:

    rails g model AssetType name:string
    rails g model Asset name:string description:text asset_type:references
    rails g model Tag name:string

Dla powiązania wiele do wielu między *Asset* i *Tag*, zgodnie
z konwencją Rails, powinniśmy dodać tabelę o nazwie *assets_tags*
(nazwy tabel w kolejności alfabetycznej):

    rails g model AssetsTags asset:references tag:references

W migracji dopisujemy *:id => false* (dlaczego? konwencja Rails)
i usuwamy zbędne *timestamps* (też wymagane przez Rails?):

    :::ruby
    class CreateAssetsTags < ActiveRecord::Migration
      def change
        create_table :assets_tags, :id => false do |t|
          t.references :asset
          t.references :tag
        end
        add_index :assets_tags, :asset_id
        add_index :assets_tags, :tag_id
      end
    end

Dopiero teraz migrujemy i usuwamy niepotrzebny model (w dowolnej
kolejności):

    rm app/models/assets_tags.rb
    rake db:migrate

Na koniec, dodajemy powiązania.

*Asset*:

    :::ruby
    class Asset < ActiveRecord::Base
      has_and_belongs_to_many :tags
      belongs_to :asset_type
    end

*Tag*:

    :::ruby
    class Tag < ActiveRecord::Base
      has_and_belongs_to_many :assets
    end

*AssetType* (dlaczego nie wystarczy krótsza nazwa *Type*):

    :::ruby
    class AssetType < ActiveRecord::Base
      has_many :assets
    end

Skorzystamy z zadania *rake* o nazwie *db:seed*
do umieszczenia danych w tabelach:

    :::ruby db/seeds.rb
    AssetType.create :name => 'Photo'
    AssetType.create :name => 'Painting'
    AssetType.create :name => 'Print'
    AssetType.create :name => 'Drawing'
    AssetType.create :name => 'Movie'
    AssetType.create :name => 'CD'

    Asset.create :name => 'Cypress', :description => 'A photo of a tree.', :asset_type_id => 1
    Asset.create :name => 'Blunder', :description => 'An action file.', :asset_type_id => 5
    Asset.create :name => 'Snap', :description => 'A recording of a fire.', :asset_type_id => 6

    Tag.create :name => 'hot'
    Tag.create :name => 'red'
    Tag.create :name => 'boring'
    Tag.create :name => 'tree'
    Tag.create :name => 'organic'

Ale wcześniej usuniemy bazę:

    rake db:drop
    rake db:schema:load

Powiązania dodamy na konsoli Rails:

    :::ruby
    a = Asset.find 1
    t = Tag.find [4, 5]
    a.tags << t
    Asset.find(2).tags << Tag.find(3)
    Asset.find(3).tags << Tag.find([1, 2])

Chcemy zbadać powiązania między powyżej wygenerowanymi modelami.
Zabadamy powiązania z konsoli Rails.

Konsola Rails:

    :::ruby
    a = Asset.find(3)
    y a
    a.name
    a.description
    a.asset_type.name
    a.tags
    a.tags.each { |t| puts t.name } ; nil
    y a.asset_type
    y a.tags
    a.tags.each { |t| puts t.name } ; nil
    aa = Asset.first
    puts aa.to_yaml

Przyjrzeć się uważnie co jest wypisywane na terminalu.
