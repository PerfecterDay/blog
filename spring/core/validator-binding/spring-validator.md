# Validator
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/validation/validator.html


Spring 提供了一个 `Validator` 接口，可用于验证对象。在验证过程中， `Validator` 能够将验证失败的结果报告给 `Errors` 对象。

`Validator` 中包含两个方法：
+ `supports(Class)` : 确定这个 `Validator` 是否支持指定 `Class` 的验证
+ `validate(Object, org.springframework.validation.Errors)` : 验证一个对象，如果有验证错误，将错误信息通过 `Errors` 返回

实现 `Validator` 相当简单，尤其当你了解Spring框架提供的 `ValidationUtils` 辅助类时。以下示例实现了 `Person` 实例的 `Validator` ：
```
public class Person {

	private String name;
	private int age;

	// the usual getters and setters...
}

public class PersonValidator implements Validator {

	/**
	 * This Validator validates only Person instances
	 */
	public boolean supports(Class clazz) {
		return Person.class.equals(clazz);
	}

	public void validate(Object obj, Errors e) {
		ValidationUtils.rejectIfEmpty(e, "name", "name.empty");
		Person p = (Person) obj;
		if (p.getAge() < 0) {
			e.rejectValue("age", "negativevalue");
		} else if (p.getAge() > 110) {
			e.rejectValue("age", "too.darn.old");
		}
	}
}
```

虽然完全可以实现一个单一的 `Validator` 类来验证丰富对象中的每个嵌套对象，但更好的做法是将每个嵌套对象类的验证逻辑封装在其专属的 `Validator` 实现中。一个简单的"丰富"对象示例是 `Customer` 类，它由两个 `String` 属性（名字和姓氏）以及一个复杂的 `Address` 对象组成。由于 `Address` 对象可独立于 `Customer` 对象使用，因此已实现独立的 `AddressValidator` 。若希望 `CustomerValidator` 复用 `AddressValidator` 类中的逻辑而不依赖复制粘贴，可通过依赖注入或在 `CustomerValidator` 内部实例化 `AddressValidator` 实现，如下例所示：
```
public class CustomerValidator implements Validator {

	private final Validator addressValidator;

	public CustomerValidator(Validator addressValidator) {
		if (addressValidator == null) {
			throw new IllegalArgumentException("The supplied [Validator] is " +
				"required and must not be null.");
		}
		if (!addressValidator.supports(Address.class)) {
			throw new IllegalArgumentException("The supplied [Validator] must " +
				"support the validation of [Address] instances.");
		}
		this.addressValidator = addressValidator;
	}

	/**
	 * This Validator validates Customer instances, and any subclasses of Customer too
	 */
	public boolean supports(Class clazz) {
		return Customer.class.isAssignableFrom(clazz);
	}

	public void validate(Object target, Errors errors) {
		ValidationUtils.rejectIfEmptyOrWhitespace(errors, "firstName", "field.required");
		ValidationUtils.rejectIfEmptyOrWhitespace(errors, "surname", "field.required");
		Customer customer = (Customer) target;
		try {
			errors.pushNestedPath("address");
			ValidationUtils.invokeValidator(this.addressValidator, customer.getAddress(), errors);
		} finally {
			errors.popNestedPath();
		}
	}
}
```

 `Validator` 也可在本地调用，用于对给定对象进行即时验证，无需涉及绑定过程。从6.1版本起，此功能通过新增的 `Errors Validator.validateObject(Object)` 方法得到简化，该方法现已默认可用，返回 `Errors` 对象：通常调用 `hasErrors()` 方法，或使用新增的 `Errors.failOnError` 方法将错误摘要转换为异常（例如： `validator.validateObject(myObject).failOnError(IllegalArgumentException::new)` ）。
```
public interface Errors {
    ....
    default <T extends Throwable> void failOnError(Function<String, T> messageToException) throws T {
		if (hasErrors()) {
			throw messageToException.apply(toString());
		}
	}
    ...
}
```