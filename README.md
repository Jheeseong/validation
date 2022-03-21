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
