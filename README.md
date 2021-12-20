# django-coupon-codes
make coupon system to your django project

lets start with first step

we need to create models that have a name and amount for coupn 

```python

class Coupon(models.Model):
    code = models.CharField(max_length=15, unique=True)
    amount = models.FloatField()

    def __str__(self):
        return self.code

```

then we need to edit the order model and add ForeignKey:

```python
class Order(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    items = models.ManyToManyField(OrderItem)
    start_date = models.DateTimeField(auto_now_add=True,null=True)
    orderd_date = models.DateTimeField(null=True)
    ordered = models.BooleanField(default=False)
    billing_address = models.ForeignKey('BillingAddress', on_delete=models.SET_NULL, blank=True, null=True)
    payment = models.ForeignKey('Payment', on_delete=models.SET_NULL, blank=True, null=True)
    # new
    coupon = models.ForeignKey('Coupon', on_delete=models.SET_NULL, blank=True, null=True)
    
    
   def __str__(self):
        return self.user.username

    def get_total(self):
        total = 0
        for order_item in self.items.all():
            total += order_item.get_final_price()
            
        # new
        if self.coupon:
            total -= self.coupon.amount
            
        return total
```

now in <code>views.py</code> lets add the function and the class view:

function to get the copon form the html template
```python
def get_coupon(request, code):
    try:
        coupon = Coupon.objects.get(code=code)
        return coupon
    except ObjectDoesNotExist:
        messages.info(request, "This coupon does not exist")
        return HttpResponseRedirect(request.META.get('HTTP_REFERER'))
```
now class view to set the coupon:

```python
class AddCouponView(View):
    def post(self, *args, **kwargs):
        form = CouponForm(self.request.POST or None)
        if form.is_valid():
            try:                    
                code = form.cleaned_data.get('code')
                order = Order.objects.get(user=self.request.user, ordered=False)
                try:
                    order.coupon = get_coupon(self.request, code)
                    order.save()
                except:
                    return redirect('core:checkout')
                messages.success(self.request, "Successfully added coupon")
                return redirect("core:checkout") # here the checkout page
            except ObjectDoesNotExist:
                messages.info(self.request, "You do not have an active order")
                return redirect("core:checkout") # here the checkout page
```

now lets create form for the coupon: <code>form.py</code>
```python
class CouponForm(forms.Form):
    code = forms.CharField(widget=forms.TextInput(attrs={
        # some attrs 
        'class':'form-control',
        'placeholder':'promocode',
        'aria-label':'Recipient\'s username',
        'aria-describedby':'basic-addon2',
    }))
```
now in views.py you must import the form and put it context for checkout def/class
examlpe:
```python
class CheckoutView(View):
  def get(self, *args, **kwargs):
        form = CheckoutForm()
        context = {
        'couponform': CouponForm(),
        }
        return render(self.request,'pages/checkout.html',context)
```

in <code>checkout.html</code>

```html
<!-- simple form with bootstrap -->
    <form class="card p-2" action="{% url 'core:add-coupon' %}" method="POST">
      {% csrf_token %}
      <div class="input-group">
          {{ couponform.code }}
          <div class="input-group-append">
          <button class="btn btn-secondary btn-md waves-effect m-0" type="submit">Redeem</button>
          </div>
      </div>
  </form>
```

# thats it you ready to go !!
