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

**ValidationItemControllerV1 - addItem()

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
