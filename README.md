# Django Dependent/Chained Dropdown List

[Watch Demonstration Video](https://www.youtube.com/CodeBand)

## Usage

models.py
```python
from django.db import models


class Country(models.Model):
    name = models.CharField(max_length=40)

    def __str__(self):
        return self.name


class City(models.Model):
    country = models.ForeignKey(Country, on_delete=models.CASCADE)
    name = models.CharField(max_length=40)

    def __str__(self):
        return self.name


class Person(models.Model):
    name = models.CharField(max_length=124)
    country = models.ForeignKey(Country, on_delete=models.SET_NULL, blank=True, null=True)
    city = models.ForeignKey(City, on_delete=models.SET_NULL, blank=True, null=True)

    def __str__(self):
        return self.name
```

urls.py
```python
from django.urls import path

from . import views

urlpatterns = [
    path('add/', views.person_create_view, name='person_add'),
    path('<int:pk>/', views.person_update_view, name='person_change'),
    
    
    path('ajax/load-cities/', views.load_cities, name='ajax_load_cities'), # AJAX
]
```

views.py
```python
from django.http import JsonResponse
from django.shortcuts import render, redirect, get_object_or_404

from .forms import PersonCreationForm
from .models import Person, City


def person_create_view(request):
    form = PersonCreationForm()
    if request.method == 'POST':
        form = PersonCreationForm(request.POST)
        if form.is_valid():
            form.save()
            return redirect('person_add')
    return render(request, 'persons/home.html', {'form': form})


def person_update_view(request, pk):
    person = get_object_or_404(Person, pk=pk)
    form = PersonCreationForm(instance=person)
    if request.method == 'POST':
        form = PersonCreationForm(request.POST, instance=person)
        if form.is_valid():
            form.save()
            return redirect('person_change', pk=pk)
    return render(request, 'persons/home.html', {'form': form})


# AJAX
def load_cities(request):
    country_id = request.GET.get('country_id')
    cities = City.objects.filter(country_id=country_id).all()
    return render(request, 'persons/city_dropdown_list_options.html', {'cities': cities})
    # return JsonResponse(list(cities.values('id', 'name')), safe=False)
```
forms.py
```python
from django import forms

from persons.models import Person, City


class PersonCreationForm(forms.ModelForm):
    class Meta:
        model = Person
        fields = '__all__'

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.fields['city'].queryset = City.objects.none()

        if 'country' in self.data:
            try:
                country_id = int(self.data.get('country'))
                self.fields['city'].queryset = City.objects.filter(country_id=country_id).order_by('name')
            except (ValueError, TypeError):
                pass  # invalid input from the client; ignore and fallback to empty City queryset
        elif self.instance.pk:
            self.fields['city'].queryset = self.instance.country.city_set.order_by('name')

```
city_dropdown_list_options.html
```html
<option value="">---------</option>
{% for city in cities %}
<option value="{{ city.pk }}">{{ city.name }}</option>
{% endfor %}
```

home.html
```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport"
          content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Dependent Dropdown in Django</title>
</head>
<body>
<h2>Person Form</h2>

<form method="post" id="personForm" data-cities-url="{% url 'ajax_load_cities' %}">
    {% csrf_token %}
    {{ form.as_p }}
    <input type="submit" value="Submit">
</form>

<script src="https://code.jquery.com/jquery-3.3.1.min.js"></script>
<script>
    $("#id_country").change(function () {
        const url = $("#personForm").attr("data-cities-url");  // get the url of the `load_cities` view
        const countryId = $(this).val();  // get the selected country ID from the HTML input

        $.ajax({                       // initialize an AJAX request
            url: url,                    // set the url of the request (= /persons/ajax/load-cities/ )
            data: {
                'country_id': countryId       // add the country id to the GET parameters
            },
            success: function (data) {   // `data` is the return of the `load_cities` view function
                $("#id_city").html(data);  // replace the contents of the city input with the data that came from the server
                /*

                let html_data = '<option value="">---------</option>';
                data.forEach(function (city) {
                    html_data += `<option value="${city.id}">${city.name}</option>`
                });
                console.log(html_data);
                $("#id_city").html(html_data);

                */
            }
        });

    });
</script>

</body>
</html>
```
