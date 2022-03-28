# validation
# v1.0 3/21
# Validation(검증)
- 컨트롤러의 중요 역할 중 하나는 HTTP 요청이 정상인지 **검증** 하는 것

**클라이언트 검증, 서버 검증**
- 클라이언트 검증은 조작 가능성이 있어 보안에 취약
- 서버만으로 검증하면, 즉각적인 고객 사용성이 부족
- 둘을 적절히 섞어 사용하되, 최종적으론 서버 검증이 필수
- API 방식 사용 시 API 스펙을 잘 정의하여 검증 오류를 API 검증 결과에 잘 남겨야 함

# Validation V1
**상품 저장 성공**
![image](https://user-images.githubusercontent.com/96407257/159284034-dac09e05-9df7-4a40-b6c1-cd6b71155a63.png)
- 사용자가 상품 등록 폼에서 정상 범위의 데이터를 입력하면, 서버에서 검증 로직이 통과하고, 상품을 저장 후 상세 화면으로 redirect

**상품 저장 검증 실패**
![image](https://user-images.githubusercontent.com/96407257/159284246-b0db54ba-7b39-4d76-8b9c-f10416371efd.png)
- 고객이 상품등록 폼에서 상품명, 가격 등의 값을 입력 혹은 검증 범위를 넘어서게 작성하였을 경우, 서버 검증 로직이 실패하도록 설계가 필수
- 실패할 경우 고객에게 다시 삼품 등록 폼을 보여주고, 잘못된 값을 알려줘야 함

**ValidationItemControllerV1 - addItem()**

    @PostMapping("/add")
    public String addItem(@ModelAttribute Item item, RedirectAttributes redirectAttributes, Model model) {
        //검증 오류 결과 보관
        Map<String, String> errors = new HashMap<>();

        if (!StringUtils.hasText(item.getItemName())) {
            errors.put("itemName", "상품 이름을 넣어주세요.");
        }
        if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
            errors.put("price", "가격은 1,000 ~ 1,000,000원 까지 가능합니다.");
        }
        if (item.getQuantity() == null || item.getQuantity() >= 9999) {
            errors.put("quantity", "수량은 최대 9,999개 까지 가능합니다.");
        }
        if (item.getQuantity() != null && item.getPrice() != null) {
            int result = item.getPrice() * item.getQuantity();
            if (result < 10000) {
                errors.put("globalError", "총 가격이 10,000 이상이여야 주문이 가능합니다. 현재 값 = " + result);
            }
        }
        // 에러가 있을 시 다시 입력 폼으로
        if (!errors.isEmpty()) {
            model.addAttribute("errors", errors);
            return "validation/v1/addForm";
        }
        // 성공 로직
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v1/items/{itemId}";
    }
    
**Map<String, String> errors = new HashMap<>();**
- 검증 오류 발생 시 오류 정보를 담아둠

**if (!StringUtils.hasText(item.getItemName()))**
- StringUtils를 통해 해당 데이터 텍스트 존재유무를 파악
- 검증 오류 발생 시 errors.put로 뷰에 오류 정보를 고객에게 전달

**if (item.getQuantity() != null && item.getPrice() != null) { 
...
errors.put("globalError", ...)}**
- 특정 필드의 범위를 넘어서는 검증 로직
- 오류 처리 시 일프 이름을 넣을 수 없으므로 globalError라는 key를 사용

**if (!errors.isEmpty())**
- 만약 검증에서 오류가 하나라도 발생 시 오류 메시지 출력을 위해 model에 errors 정보를 담고 입력 폼이 있는 뷰 템플릿으로 전달

**HTML**
**글로벌 오류 메시지**
- th:if="${errors?.containsKey('globalError')}"
- class="field-error" th:text="${errors['globalError']}
- errors에 내용이 있을 때 globalError을 텍스트로 출력, th:if 조건 만족 시 출력
- errors?는 errors가 null일 때 nullpointException 대신 null을 반환

**필드 오류 처리**
- th:classappend="${errors?.containsKey('itemName')} ? 'field-error': _
- classappend를 사용해서 오류가 존재할 경우 field-error 클래스 정보를 더하여 출력

**문제점**
- 뷰 템플릿에서 중복처리가 다수 존재
- 타입 오류 처리가 안됨. Item.price, quantity 경우 Integer이므로 문자 타입으로 설정이 불가능
- 문자 타입이 들어올 시 오류 발생
- price, quantity는 문자 보관이 불가능
- 입력 값 역시 별도의 관리 및 저장이 

# v1.1 3/22
# Validation V2
**ValidationItemContorollerV2 - addItemV1**  
**필드오류 - fieldError**

    if (!StringUtils.hasText(item.getItemName())) {
            bindingResult.addError(new FieldError("item", "itemName", "상품 이름을 넣어주세요."));
        }
        
- FieldError 생성자 : public FieldError(String objectName, String field, String defaultMessage) {}
- objectName : @ModelAttribute 이름
- field : 오류가 발생한 필드 이름
- defaultMessage : 오류 기본 메시지
- 필드에 오류 시 fieldError 객체를 생성해서 bindingResult에 저장

**글로벌 오류 - ObjectError**

    if (result < 10000) {
                bindingResult.addError(new ObjectError("item", "총 가격이 10,000 이상이여야 주문이 가능합니다. 현재 값 = " + result));
            }
            
- ObjectError 생성자 요약 : public ObjectError(String objectName, String defaultMessage) {}
- objectName : @ModelAttribute의 이름
- defaultMessage : 오류 기본 메시지
- 특정 필드를 넘어서는 오류 발생 시 ObjectError 객체를 생성해서 bindinResult에 저장

**HTML**
- 타임리프는 스프링의 BindingResult를 활용해서 검증 오류를 표현하는 기능을 제공
- #fields : #fields 로 BindingResult 가 제공하는 검증 오류에 접근
  - th:if="${#fields.hasGlobalErrors()}
  -  class="field-error" th:each="err : ${#fields.globalErrors()}" 
- th:errors : 해당 필드에 오류가 있는 경우에 태그를 출력, th:if 의 편의 버전
  - class="field-error" th:errors="*{itemName}"
- th:errorclass : th:field 에서 지정한 필드에 오류가 있으면 class 정보를 추가
  - th:errorclass="field-error" class="form-control" placeholder="이름을 입력하세요"

# BindingResult
- 스프링이 제공하는 검증 오류를 보관하는 객체, 오류 발생 시 BindingResult에 보관
- @ModelAttribute에 데이터 바인딩 시 오류가 발생해도 컨트롤러를 정상 호출

**BindingResult에 검증 오류를 적용하는 3가지 방법**
- @ModelAttribute의 객체에 타입 오류 등으로 바인딩이 실패하는 경우 스프링이 fieldError 생성해서 BindingResult에 넣음
- 개발자가 직접 넣음
- validator 사용

**타입 오류 확인**
- Integer 부분에 문자를 입력하여 타입이 다를 경우 BindingResult를호출하고 값을 확인
- BindingResult는 검증할 대상 바로 다음에 와야함. 예를 들어 @ModelAttribute Item item, 다음 BindingResult가 와야 함

**BindingResult 와 Errors
- BindingResult 는 인터페이스, Errors 인터페이스를 상속
- 실제 넘어오는 구현체는 BeanPropertyBindingResult 라는 것인데, 둘다 구현하고 있으므로BindingResult 대신에 Errors 를 사용하는 것도 가능
- 그러나 Errors 인터페이스는 단순한 오류 저장과 조회 기능만 제공
- BindingResult는 추가적인 기능을 제공

# v1.2 3/23
# 오류코드와 메시지 처리1
# FieldError, ObjectError
**FieldError 생성자**

    public FieldError(String objectName, String field, @Nullable Object rejectedValue, boolean bindingFailure, @Nullable String[] codes, @Nullable Object[] arguments, @Nullable String defaultMessage)
    
- objectName : 오류가 발생한 객체 이름
- field : 오류 필드
- rejectedValue : 사용자가 입력한 값(거절된 값)
- bindingResult : 타임 오류 같은 바인딩 실패, 검증 실패인지 구분 값
- codes : 메시지 코드
- arguments : 메시지에서 사용하는 인자
- defaultMessage : 기본 오류 메시지

**오류 발생 시 사용자 입력 값 보관 및 유지**
- 사용자의 입력 데이터가 컨트롤러의 @ModelAttribute에 바인딩되는 시점에 오류 발생 시 입력 값 유지가 힘듦
- fieldError는 오류 발생 시 사용자 입력 값을 저장하는 기능을 제공
- rejectedValue가 오류 발생 시 입력 값을 저장하는 필드

**HTML 타임리프의 사용자 입력 값 유지
- th:field="* {price}"
  - fieldError에서 보관한 값을 사용해서 값을 출력

**스프링의 바인딩 오류 처리**
- 타입 오류로 바인딩 실패 시 스프링은 FieldError를 생성하면서 사용자 입력 값을 주입
- 해당 오류를 BindingResult에 담아서 컨트롤러를 호출하기 떄문에 실패 시에도 사용자 오류 메시지가 정상 출력

**Errors 메시지 파일 생성**
- errors.properties라는 별도 파일을 생성
- application.properties -> spring.messages.basename=messages,errors 삽입
- errors.properties 추가 -> required.item.itemName=상품 이름은 필수입니다., range.item.price=가격은 {0} ~ {1} 까지 허용합니다. 등등

**코드 변경**

    new FieldError("item", "price", item.getPrice(), false, new String[]
    {"range.item.price"}, new Object[]{1000, 1000000}
    
- codes : range.item.price를 사용해서 메시지 코드를 지정
- arguments : Object[]{1000, 1000000}를 사용해서 사용 코드의 {0},{1}에 치환 값을 전달

# v1.3 3/24
# 오류코드와 메시지 처리2
# rejectValue(), jeject()
- BindingResult에서 제공하는 rejectValue(), reject()를 사용하면 fieldError, ObjectError를 생성하지 않고 검증 오류가 가능

**RejectValue**

    void rejectValue(@Nullable String field, String errorCode,@Nullable Object[] errorArgs, @Nullable String defaultMessage);
    
- field : 오류 필드 명
- errorCode : 오류 코드(메시지에 등록된 오류코드가 아닌 messageResolver를 위한 오류 코드)
- errorArgs : 오류 메시지에서 {0} 치환하기 위한 값
- defaultMessage : 오류 메시지를 찾을 수 없을 때 사용하는 기본 메시지
- ex) bindingResult.rejectValue("price", "range", new Object[]{1000, 1000000}, null)
- bindingREsult는 어떤 객체를 대상으로 검증하는지 target을 이미 알고 있어서 targer(item)에 대한 정보가 없이도 검증이 가능

**reject()**

    void reject(String errorCode, @Nullable Object[] errorArgs, @Nullable String defaultMessage);
    
# 오류 코드와 메시지 처리3
- 오류 코드에서 required.item.itemName 으로 자세하게 조회가 가능할 수도 있고, required로 단순하게 조회도 가능
- 세밀하게 작성해야하는 경우 세밀한 내용이 적용되도록 메시지에 단계를 둠
- error.properties

      #Level1
      required.item.itemName: 상품 이름은 필수 입니다.
      #Level2
      required: 필수 값 입니다.
      
- 더 자세한 경로의 내용이 우선이고, 없을 경우 단순한 경로의 내용이 조회
- 스프링 MessageCodesResolver로 이러한 기능을 지원

# v1.4 3/25
# DefaultMessageCodesResolver 기본 메시지 생성
**동작 방식**
- rejectValue(), reject()는 내부에서 MessageCodesResolver을 사용
- fieldError, ObjectError의 생성자를 보면, 여러 오류 코드 가지는 것이 가능
- MessageCodesResolver를 통해서 생성된 오류 코드를 순서대로 보관

**ex) FieldError rejectValue("itemName", "required")**
- 다음 4가지 오류 코드를 자동으로 생성
  - required.item.itemName
  - required.itemName
  - required.java.lang.String
  - required

**ex) ObjectError reject("totalPriceMin")**
- 다음 2가지 오류 코드를 자동으로 생성
  - totalPriceMin.item
  - totalPriceMin

# 오류 코드 관리 전략
- 구체적인 것을 우선 조회 후, 덜 구제적인 것으로 동작
  - MessageCodesResolver 는 required.item.itemName 처럼 구체적인 것을 우선 조회 후, required 처럼 덜 구체적인 것을 가장 나중에 생성
- 복잡하게 사용하는 이유
  - 모든 오류 코드에 대해서 각각 다 정의 시 관리가 힘듦
  - 중요도에 따라 분리 후 범용성 있는 required 같은 메시지를 사용하거나, 더 구체적으로 작성하여 사용하는 것이 효과적

# 스프링이 직접 만든 오류 메시지 처리
- 검증 오류 코드는 **개발자가 설정한 오류 코드**와 **스프링이 직점 검증 오류에 추가** 한 경우 2가지로 분리 가능
- 타입 오류 시 typeMismach 오류 메시지 코드가 생성
- error.properties에 **typeMismatch.java.lang.Integer=숫자를 입력해주세요.** 추가 시 오류 메시지가 변경

# v1.5 3/26
# Validator 분리 - V1
**ItemValidator 생성 후 분리**

    @Component
    @Slf4j
    public class ItemValidator implements Validator {
        @Override
        public boolean supports(Class<?> clazz) {
            return Item.class.isAssignableFrom(clazz);
        }

        @Override
        public void validate(Object target, Errors errors) {
        
            Item item = (Item) target;

            log.info("objectName ={}", errors.getObjectName());

            if (!StringUtils.hasText(item.getItemName())) {
                errors.rejectValue("itemName", "required");
            }
            if (item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
                errors.rejectValue("price", "range", new Object[]{1000,1000000}, null);
            }
            if (item.getQuantity() == null || item.getQuantity() >= 9999) {
                errors.rejectValue("quantity","max",new Object[]{9999}, null);
            }
            if (item.getQuantity() != null && item.getPrice() != null) {
                int result = item.getPrice() * item.getQuantity();
                if (result < 10000) {
                    errors.reject("totalPriceMin", new Object[]{10000, result}, null);
                }
            }
        }
    }
    
**Interface validator**

    public interface Validator {
      boolean supports(Class<?> clazz);
      void validate(Object target, Errors errors);
    }
    
- supports() {} : 해당 검증기를 지원하는 여부 확인
- validate(Object target, Errors errors : 검증 대상 객체와 bindingResult

# Validator 분리 - v2
- 체계적으로 검증 기능 도입을 위해 스프링이 Validator 인터페이스를 별도로 제공

**WebDataBinder**
- 스프링의 파라미터 바인딕의 역할과 검증 기능이 포함

**ValidationITemControllerV2**

    @InitBinder
    public void init(WebDataBinder dataBinder) {
      log.info("init binder {}", dataBinder); 
      dataBinder.addValidators(itemValidator);
    }
    
- WebDataBinder에 검증기 추가 시 해당 컨트롤러에서는 검증기를 자동으로 적용
- @InitBinder : 해당 컨트롤러에만 영향

**@Validated 적용**

    public String addItemV6(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {
    ...
    }
    
- @Validated는 검증기를 실핼하라는 애노테이션
- 이 애노테이션이 webDataBinder에 등록한 검증기를 찾아서 실행
- supports() 는 여러 검증기 사용 시 구분하기 위해 사용
- ex) supports(Item.class) 호출 -> 결과 true -> itemValidator의 validate() 가 호출

# v1.6 3/27
# Validation - BeanValidation
**BeanValidation**
- 특정한 구현체가 아닌 Bean Validation2.0(JSR-380)라는 기줄 표준
- 검증 애노테이션과 여러 인터페이스의 모음

**Item**

    public class Item {
     
      private Long id;
     
      @NotBlank
      private String itemName;
 
      @NotNull
      @Range(min = 1000, max = 1000000)
      private Integer price; @NotNull
 
      @Max(9999)
      private Integer quantity;
      ...
    }
   

# Bean Validation 사용
**Bean Validation 의존관계 추가**

**build.gradle**

    implementation 'org.springframework.boot:spring-boot-starter-validation'

**검증 에노테이션**
- @NotBlank : 빈값 + 공백만 있는 경우를 허용하지 않음
- @NotNull : null 을 허용하지 않음
- @Range(min = 1000, max = 1000000) : 범위 안의 값
- @Max(9999) : 최대 9999까지만 허용


**BeanValidationTest**

    @Test
    void beanValidation() {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        Validator validator = factory.getValidator();

        Item item = new Item();
        item.setItemName("  ");
        item.setPrice(0);
        item.setQuantity(10000);

        Set<ConstraintViolation<Item>> validate = validator.validate(item);
        for (ConstraintViolation<Item> itemConstraintViolation : validate) {
            System.out.println("validate = " + validate);
            System.out.println("itemConstraintViolation.getMessage() = " + itemConstraintViolation.getMessage());
        }
    }
    
**검증기 생성**

    ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
    Validator validator = factory.getValidator();
    
- 스프링과 통합 시에는 코드를 작성하지 않음

**검증 실행**

    Set<ConstraintViolation<Item>> validate = validator.validate(item);
    
- 검증 대상을 검증기에 넣고 결과를 받음
- set에는 contraintViolation이라는 검증 오류가 담기고, 결과가 비어있을 경우 오류가 없는 

# v1.7 3/28
# Bean Validation - 스프링 적용

**ValidationItemControllerV3**

    @PostMapping("/add")
    public String addItemV1(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {
        if (item.getQuantity() != null && item.getPrice() != null) {
            int result = item.getPrice() * item.getQuantity();
            if (result < 10000) {
                bindingResult.reject("totalPriceMin", new Object[]{10000, result}, null);
            }
        }

        // 에러가 있을 시 다시 입력 폼으로
        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "validation/v3/addForm";
        }
        // 성공 로직
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v3/items/{itemId}";
    }
    
**스프링MVC에 Bean Validation 사용**
- spring-boot-starter-validation 라이브러리를 넣을 시 스트링 부트가 자동으로 Bean Validator를 인지하고 스프링에 통합

**스프링 부트의 자동 글로벌 validator 등록**
- LocalValidatorFactoryBean을 글로벌 validator로 등록
- 이 validator은 @NotNull 같은 애노테이션을 보고 검증을 수행
- 검증 오류 발생 시 FieldError, ObjectError를 생성해서 BindingResult에 저장

**검증 순서**
1. @ModelAttribute 각각의 필드에 타입 변환 시도(실패 시 typeMismatch로 FieldError 추가)
2. Validator 적용

**바인딩에 성공한 필드만 Bean Validation 적용**
- BeanValidator는 바인딩에 실패한 필드는 적용 X
- 타입 변환에 성곡해서 바인딩에 성공한 필드여야 적용
- ex) itemName에 문자 "A' 입력 -> 타입 변환 성공 -> itemName 필드에 BeanValidation 적용
- ex) price에 문자 'A' 입력 -> 숫자 타입 변환 실패 -> typeMismatch FieldError 추가 -> price 필드는 BeanValidation 적용 X
