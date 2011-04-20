#### {% title "FormBuilder v. simple_form gem" %}

Dlaczego? Kiedy *simple_form* nie wystarcza?

Wspomnieć o gemie *ActiveScaffold*. Jaki problem on rozwiązuje?

Na początek instalujemy rozszerzenie View Source Chart do Firefoksa:

    https://addons.mozilla.org/en-US/firefox/addon/655/

Nieco dokumentacji jest tutaj:

    http://andrewtimberlake.com/blog/how-to-make-a-custom-form-builder-in-rails

Chcemy tak przebudować formularz, aby w wypadku
błędów walidacji wyświetlało się coś takiego:

    Name: [          ] can't be blank

co po tłumaczeniu ma wyglądać tak:

    Nazwa: [          ] nie może być puste

Poniższy kod, z Rails Guides,
[8.3 Customizing the Error Messages HTML](http://edgeguides.rubyonrails.org/active_record_validations_callbacks.html#creating-custom-validation-methods):

    ActionView::Base.field_error_proc = Proc.new do |html_tag, instance|
      if instance.error_message.kind_of?(Array)
        %(#{html_tag}<span class="validation-error">&nbsp;
          #{instance.error_message.join(',')}</span>)
      else
        %(#{html_tag}<span class="validation-error">&nbsp;
          #{instance.error_message}</span>)
      end
    end

nie działa tak jak trzeba, ponieważ daje coś takiego:

    Name: can't be blank [          ] can't be blank

Działający kod:

    ActionView::Base.field_error_proc = Proc.new do |html_tag, instance|
      if html_tag =~ /type="hidden"/ || html_tag =~ /<label/
        html_tag
      else
        "<span class='field_error'>#{html_tag}</span>".html_safe +
        "<span class='error_message'>#{[instance.error_message].join(', ')}</span>".html_safe
      end
    end

Odstępy dodajemy w CSS.

Kod ten umieszczamy w pliku *config/initializers/change_field_error.rb*.

Zaczynamy od takiego kodu:

    <%= form_for(@task) do |f| %>
      <% if @task.errors.any? %>
      <div id="error_explanation">
        <h2><%= t('activerecord.errors.template.header') %></h2>
        <p><%= t('activerecord.errors.template.body') %></p>
        <!-- usuwamy – nie będzie już potrzebne -->
        <ul>
          <% @task.errors.full_messages.each do |msg| %>
          <li><%= msg %></li>
          <% end %>
        </ul>
        <!-- usuwamy aż dotąd -->
      </div>
      <% end %>

      <p><%= f.label :name %> <%= f.text_field :name %></p>

      <div class="actions">
        <%= f.submit %>
      </div>
    <% end %>


## Refaktoryzacja

Na początek usuniemy ten nadmiarowy kod w szablonie *_form*:

    <p><%= f.label :name %> <%= f.text_field :name %></p>

Zamienimy go na:

    <p><%= f.text_field :name %></p>

a generowanie etykiety przeniesiemy do LabeledFormBuilder
(plik *app/form_builders/labeled_form_builder.rb*, katalog dodajemy
do LOAD_PATH).


### Pierwsza wersja

W pliku *config/initializers/change_field_error.rb* wpisujemy:

    class LabeledFormBuilder < ActionView::Helpers::FormBuilder
      def text_field(field_name, *args)
        "<b>custom text field</b>".html_safe
      end
    end

Sprawdzamy czy to działa. Nie.

Należy jeszcze dopisać do formularza:

    <%= form_for(@task, :builder => LabeledFormBuilder) do |f| %>

Teraz poprawiamy kod generujący html dla *text_field*:

    def text_field(field_name, *args)
      @template.content_tag(:span, label(field_name) + super(field_name, *args))
    end

Jeśli, na przykład, pole *collection_select*:

    f.collection_select :some_id, Tag.all, :id, :name

ma też otrzymać automatycznie wygenerowany *label*, to kod będzie taki:

    def collection_select(field_name, *args)
      @template.content_tag(:span, label(field_name) + super)
    end

Jak widać ciało obu funkcji jest takie samo. Dlatego postąpimy tak:

    %w[text_field text_area collection_select password_field].each do |method_name|
      define_method(method_name) do |field_name, *args|
        @template.content_tag(:span, label(field_name) + super(field_name, *args))
      end
    end

Wyjątki, etykieta po elemencie formularza, ns przykład
(trochę naciągany przykład – przydałoby się nowe pole w modelu):

    <%= f.text_area :name %> <= f.label :name %>

Też chcielibyśmy wpisywać tak (na razie bez dodatkowego argumentu):

    <%= f.text_area :name %>

Teraz kod wyglada tak (przestawiamy kolejność label z super):

    def text_area(field_name, *args)
      @template.content_tag(:span, super(field_name, *args) + label(field_name))
    end


### Obsługa dodatkowych argumentów: *label*

Dodajemy argument *label*:

    <%= f.text_field :name, :label => "Upływa termin oddania projektu!" %>

Kod:

    def text_field(field_name, *args)
      options = args.extract_options!
      @template.content_tag(:span, label(field_name, options[:label]) + super(field_name, *args))
    end

A tak będzie wyglądał powyższy kod po „liftingu”:

    def text_field(field_name, *args)
      @template.content_tag(:span, field_label(field_name, *args) + super(field_name, *args))
    end

    private

    def field_label(field_name, *args)
      options = args.extract_options!
      label(field_name, options[:label])
    end


### Obsługa dodatkowych argumentów

Taki przykład, zmiana etykiety + dodanie atrybutu CSS label_class="required":

    <p><%= f.text_field :name, :label => 'login lub email', :required => true %></p>

Kod:

    def field_label(field_name, *args)
      options = args.extract_options!
      options[:label_class] = "required" if options[:required]
      label(field_name, options[:label], :class => options[:label_class])
    end

Kod do poprawki, ponieważ element *input* teraz to:

    <input id="task_name" label_class="required" name="task[name]" required="required"

Jakoś trzeba użyte argumenty usunąć z hasza.

Hakowanie w czystej postaci!
W części **private** nadpisujemy metodę Rails:

    # fix the problem
    def objectify_options(options)
      super.except(:label, :required, :label_class)
    end


## Finishing touches…

Można podmienić oryginalny FormBuilder na LabeledFormBuilder:

    ActionView::Base.default_form_builder = LabeledFormBuilder

w jakimś pliku w katalogu *initializers*, albo lepsze rozwiązanie:

    <%= labeled_form_for(@task) do |f| %>

i definiujemy *labeled_form_for* w *ApplicationHelper*:

    module ApplicationHelper
      def labeled_form_for(*args, &block)
        options = args.extract_options!.merge(:builder => LabeledFormBuilder)
        form_for(*(args + [options]), &block)
      end
    end


## Error messages — inline & usuwamy extra div

To już załatwia poprawiony kod.
Ale teraz można to zrobić inaczej: wypisujemy wszystkie błędy walidacji zamiast jednego
(co niekoniecznie jest dobrym pomysłem):

class LabeledFormBuilder < ActionView::Helpers::FormBuilder
  %w[text_field text_area].each do |method_name|
    define_method(method_name) do |field_name, *args|
      @template.content_tag(:span,
         field_label(field_name, *args) + super(field_name, *args) + field_error(field_name))
    end
  end
  ...
  private
  ...
  def field_error(field_name)
    if object.errors[field_name].any?
      @template.content_tag(:span, [object.errors[field_name]].join(', '), :class => 'error_message')
    else
      ''
    end
  end

Z formularza usuwamy generowanie błędów w osobnym DIV.

Pozostaje usunąć extra div z komunikatów o błędach.

W pliku *config/initializers/change_field_error.rb* wpisujemy:

    ActionView::Base.field_error_proc = Proc.new do |html_tag, instance_tag|
      "<span class='field_error'>#{html_tag}</span>".html_safe
    en


## TODO

Podrasować CSS.
