---
layout: post
title: Springboot Junit5
category: [junit5]
tags: [springboot, junit5]
redirect_from:

- /2022/02/03/

---

## junit5
Junit 4는 단일 모듈로 구성되어 있고 JUnit 5는 Junit Jupiter, JUnit plateform,  JUnit Vintage모듈로 구성된다.    
Junit Platform : 테스트들을 실행하기 위한 뼈대이다. 테스트를 발견하고 테스트 계획을 생성하는 TestEngine 인터페이스를 가지고 있다. Platform은 TestEngine을 통해서 테스트를 발견하고, 실행하고, 결과를 보고한다. 또한 콘솔출력, 각종 IDE들의 연동을 보조하는 역할을 한다.  
Junit Jupiter : TestEngine의 실제 구현체는 별도 모듈의 역할을 한다. 모듈 중 하나가 jupiter-engine이다. 이 모듈은 jupiter-api를 사용해서 작성한 테스트 코드를 발견하고 실행한다.    
Junit Vintage : 기존에 JUnit 4 버전으로 작성한 테스트 코드를 실행할 때에는 vintage-engine 모듈을 사용한다.  

## Springboot Junit5 단위테스트
그동안 Controller, Service, Repository(mapper) 단위테스트를 할 때 SpringBootTest 어노테이션을 붙여서 단위테스트를 진행했었다. 최근에 TDD에 대한 정리를 하다가 Junit5에 대한 정리를 해봐야 겠다는 생각을 하게 되었고 오랜만에 노트를 작성하게 되었다. 오늘은 Controller, Service, Repository 별로 단위테스트를 위한 나만의 최적화를 찾아보려고 한다.  

## JPA Repository Junit5 테스트
```java
import com.sisipapa.junit.domain.Item;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

//@TestPropertySource(locations = "classpath:application-test.yml")
//@ActiveProfiles("test")
//@DataJpaTest
@ExtendWith(MockitoExtension.class)
class ItemRepositoryTest {

//    @Autowired
//    private ItemRepository itemRepository;

    //    @MockBean
    @Mock
    private ItemRepository itemRepository;


    @Test
    void save(){
        // given
        final Item item = Item.builder().name("item1").description("아이템1입니다.").build();

        when(itemRepository.save(any())).thenReturn(item);

        // when
        final Item saveItem = itemRepository.save(new Item());

        // then
        verify(itemRepository).save(any());
        assertEquals(saveItem.getName(), "item1");
        assertEquals(saveItem.getDescription(), "아이템1입니다.");
    }
}
```   

## Mapper Junit5 테스트
```java
import com.sisipapa.junit.domain.dto.ProductDto;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.mockito.junit.jupiter.MockitoExtension;
import org.mybatis.spring.boot.test.autoconfigure.MybatisTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class ProductMapperTest {

    @Mock
    private ProductMapper productMapper;


    @BeforeEach
    void setUp() {
    }

    @Test
    void findAll() {
        // given
        List<ProductDto> mockProducts = List.of(
                ProductDto.of().idx(1L).name("베베숲 물티슈").description("베베숲 물티슈 설명").build()
                ,ProductDto.of().idx(2L).name("여름 토퍼").description("여름 토퍼 설명").build()
                ,ProductDto.of().idx(3L).name("페이크 삭스").description("페이크 삭스 설명").build()
                ,ProductDto.of().idx(4L).name("우산").description("우산 설명").build()
        );
        when(productMapper.findAll()).thenReturn(mockProducts);

        // when
        List<ProductDto> resultProducts = productMapper.findAll();

        // then
        verify(productMapper).findAll();
        assertThat(mockProducts.get(0).getName()).isEqualTo(resultProducts.get(0).getName());
    }

    @Test
    void findByName() {
        // given
        String name = "베베숲 물티슈";
        when(productMapper.findByName(name)).thenReturn(ProductDto.of().idx(1L).name("베베숲 물티슈").description("베베숲 물티슈 설명").build());

        // when
        ProductDto mockProductDto = productMapper.findByName(name);

        //then
        verify(productMapper).findByName(name);
        assertThat(mockProductDto.getName()).isEqualTo("베베숲 물티슈");
        assertThat(mockProductDto.getDescription()).isEqualTo("베베숲 물티슈 설명");

    }

    @Test
    void insertProduct() {
        // given
        ProductDto dto = ProductDto.of().name("신규1").description("신규1 설명").build();
        when(productMapper.insertProduct(dto)).thenReturn(1);

        // when
        int cnt = productMapper.insertProduct(dto);

        // then
        verify(productMapper).insertProduct(dto);
        assertThat(cnt).isEqualTo(1);
    }

    @Test
    void updateProduct() {
        // given
        ProductDto dto = ProductDto.of().idx(1L).name("베베숲 물티슈 >> 수정").description("베베숲 물티슈 설명 >> 수정").build();
        when(productMapper.updateProduct(dto)).thenReturn(1);

        // when
        int cnt = productMapper.updateProduct(dto);

        // then
        verify(productMapper).updateProduct(dto);
        assertThat(cnt).isEqualTo(1);
    }

    @Test
    void deleteProduct() {
        // given
        Long idx = 1L;
        when(productMapper.deleteProduct(idx)).thenReturn(1);

        // when
        int cnt = productMapper.deleteProduct(idx);

        // then
        verify(productMapper).deleteProduct(idx);
        assertThat(cnt).isEqualTo(1);
    }
}
```  

### @ExtendWith(MockitoExtension.class)  
- Spring에서 Mockito를 사용하기 위해서는 @ExtendWith(MockitoExtention.class) 를 테스트 클래스 상단에 지정해주면 된다.   
- @Mock 어노테이션을 통해 간단하게 Mock 객체를 만들어 사용할 수 있다.  

### @DataJpaTest  
- @DataJpaTest 어노테이션은 Full-Auto Config를 하지 않고 JPA 관련 테스트 설정을 로드한다.   
- @DataJpaTest는 기본적으로 @Entity 어노테이션이 적용된 클래스를 스캔하여 스프링 데이터 JPA 저장소를 구성한다.      

### @Mock  
- Mock은 가짜객체를 만드는데 스프링빈에 등록이 안되는 객체이다.   
- 스프링 컨테이너가 DI를 하는 방식이 아니라 객체생성시 생성자에 Mock객체를 직접 주입한다.   
- 스프링을 띄우지 않으므로 MockBean을 사용할때보다 빠르다.  

### @MockBean  
- MockBean은 가짜 Bean을 스프링에 등록된다.   
- 스프링 컨테이너가 기존에 갖고있는 Bean객체는 MockBean객체로 치환되어 DI된다.  

### when~thenReturn VS given~willReturn  
- BDDMockito가 제공하는 기능과 Mockito가 제공하는 기능은 동일하다.  
- BDD라는 것을 테스트 코드에 도입할 때 기존의 Mockito가 가독성을 해치기 때문에 이를 해결하기 위해 기능은 같지만 이름만 다른 클래스라고 생각해도 될 것 같다.  

### Mockito.verify
- verify(mock).method(param) : 해당 Mock Object 의 메소드를 호출했는지 검증 ex) verify(mock).get(0);  
- verify(mock, times(wantedNumberOfInvocations)).method(param) : 해당 Mock Object 의 메소드가 정해진 횟수만큼 호출됬는지 검증 ex) verify(mock, times(2)).get(anyInt());  
- verify(mock, atLeast(minNumberOfInvocations)).method(param) : 해당 Mock Object 의 메소드가 최소 정해진 횟수만큼 호출됬는지 검증 ex) verify(mock, atLeast(1)).get(anyInt());  
- verify(mock, atLeastOnce()).method(param) : 해당 Mock Object 의 메소드가 최소 한번 호출됬는지 검증 ex) verify(mock, atLeastOnce()).get(anyInt());  
- verify(mock, atMost(maxNumberOfInvocations)).method(param) : 해당 Mock Object 의 메소드가 정해진 횟수보다 적게 호출됬는지 검증 ex) verify(mock, atMost(2)).get(anyInt());  
- verify(mock, never()).method(param) : 해당 Mock Object 의 메소드가 호출이 안됬는지 검증 ex) verify(mock, never()).get(2);

> JPA에서 실제 데이터베이스와 연결해서 단위테스트를 해보고 싶은 경우 @DataJpaTest @Autowired private ItemRepository itemRepository 주석을 풀고 적용할 수 있다.   

## Service Junit5 테스트
@InjectMocks 어노테이션을 사용해서 @Spy,@Mock 어노테이션이 붙은 객체들을 주입시켜준다. 아래 예제에서는 ItemRepository,ProductMapper 객체에 @Mock 어노테이션을 붙이고 ItemProductService 객체에 @InjectMocks 어노테이션을 붙여 Mock을 주입해 주었다.  

```java
import com.sisipapa.junit.domain.Item;
import com.sisipapa.junit.domain.dto.ProductDto;
import com.sisipapa.junit.mapper.ProductMapper;
import com.sisipapa.junit.repository.ItemRepository;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
public class ItemProductServiceTest {

    @Mock
    ItemRepository itemRepository;

    @Mock
    ProductMapper productMapper;

    @InjectMocks
    ItemProductService itemProductService;

    @Test
    @DisplayName("ITEM 테이블 등록")
    void itemSave(){
        // given
        Item mockItem = Item.builder().name("test-item1").description("test-item1-description").build();
        when(itemRepository.save(mockItem)).thenReturn(Item.builder().idx(5L).name("test-item1").description("test-item1-description").build());

        // when
        Item resultItem = itemProductService.itemSave(mockItem);

        // then
        verify(itemRepository).save(mockItem);
        assertThat(resultItem.getName()).isEqualTo(mockItem.getName());

    }

    @Test
    @DisplayName("Product 테이블 전체 조회")
    void findAll(){
        // given
        List<ProductDto> mockProducts = List.of(
                ProductDto.of().idx(1L).name("베베숲 물티슈").description("베베숲 물티슈 설명").build()
                ,ProductDto.of().idx(2L).name("여름 토퍼").description("여름 토퍼 설명").build()
                ,ProductDto.of().idx(3L).name("페이크 삭스").description("페이크 삭스 설명").build()
                ,ProductDto.of().idx(4L).name("우산").description("우산 설명").build()
        );
        when(productMapper.findAll()).thenReturn(mockProducts);

        // when
        List<ProductDto> resultProducts = itemProductService.findAll();

        // then
        verify(productMapper).findAll();
        assertThat(resultProducts.size()).isEqualTo(mockProducts.size());
    }

    @Test
    @DisplayName("Product 테이블 Name컬럼으로 조회")
    void findByName(){
        // given
        String name = "베베숲 물티슈";
        ProductDto mockProductDto = ProductDto.of().idx(1L).name("베베숲 물티슈").description("베베숲 물티슈 설명").build();
        given(productMapper.findByName(name)).willReturn(mockProductDto);

        // when
        ProductDto resultProductDto = itemProductService.findByName(name);

        // then
        verify(productMapper).findByName(name);
        assertThat(mockProductDto).isEqualTo(resultProductDto);
        assertThat(mockProductDto.getIdx()).isEqualTo(resultProductDto.getIdx());
        assertThat(mockProductDto.getName()).isEqualTo(resultProductDto.getName());
        assertThat(mockProductDto.getDescription()).isEqualTo(resultProductDto.getDescription());
    }

    @Test
    @DisplayName("Product 테이블 idx 필드로 조회")
    void findByIdx(){
        // given
        Long idx = 2L;
        ProductDto mockProductDto = ProductDto.of().idx(idx).name("여름 토퍼").description("여름 토퍼 설명").build();
        when(productMapper.findByIdx(idx)).thenReturn(mockProductDto);

        // when
        ProductDto resultProductDto = itemProductService.findByIdx(idx);

        // then
        verify(productMapper).findByIdx(idx);
        assertThat(mockProductDto).isEqualTo(resultProductDto);
        assertThat(mockProductDto.getIdx()).isEqualTo(resultProductDto.getIdx());
        assertThat(mockProductDto.getName()).isEqualTo(resultProductDto.getName());
        assertThat(mockProductDto.getDescription()).isEqualTo(resultProductDto.getDescription());
    }

    @Test
    @DisplayName("Product 테이블 등록")
    void insertProduct(){
        // given
        ProductDto mockProductDto = ProductDto.of().idx(5L).name("상품5").description("상품5").build();
        given(productMapper.insertProduct(mockProductDto)).willReturn(1);

        // when
        int cnt = itemProductService.insertProduct(mockProductDto);

        // then
        verify(productMapper).insertProduct(mockProductDto);
        assertThat(1).isEqualTo(cnt);
    }

    @Test
    @DisplayName("Product 테이블 수정")
    void updateProduct(){
        // given
        ProductDto mockProductDto = ProductDto.of().idx(1L).name("베베숲 물티슈 > 수정").description("베베숲 물티슈 설명 > 수정").build();
        when(productMapper.updateProduct(mockProductDto)).thenReturn(1);

        // when
        int cnt = itemProductService.updateProduct(mockProductDto);

        // then
        verify(productMapper).updateProduct(mockProductDto);
        assertThat(1).isEqualTo(cnt);
    }

    @Test
    @DisplayName("Product 테이블 삭제")
    void deleteProduct(){
        // given
        Long idx = 4L;
        when(productMapper.deleteProduct(idx)).thenReturn(1);

        // when
        int cnt = itemProductService.deleteProduct(idx);

        // then
        verify(productMapper).deleteProduct(idx);
        assertThat(1).isEqualTo(cnt);
    }
}
```  

### @InjectMock
@InjectMock은 DI를 @Mock이나 @Spy로 생성된 mock 객체를 자동으로 주입해주는 어노테이션이다.  

## Controller Junit5 테스트
```java
import com.sisipapa.junit.domain.Item;
import com.sisipapa.junit.domain.dto.ProductDto;
import com.sisipapa.junit.service.ItemProductService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.ResultActions;

import java.util.List;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;


@WebMvcTest(ItemProductController.class)
public class ItemProductControllerTest {

    @Autowired
    MockMvc mvc;

    @MockBean
    ItemProductService itemProductService;

    @BeforeEach
    public void setUp() {
//        mvc = MockMvcBuilders.standaloneSetup(new ItemProductController(itemProductService))
//                .addFilters(new CharacterEncodingFilter("UTF-8", true)) // utf-8 필터 추가
//                .build();
    }


    @Test
    void itemSave() throws Exception {

        // given
        Item mockItem = Item.builder()
                .idx(1L)
                .name("신상품1")
                .description("신상품1 설명")
                .build();
        when(itemProductService.itemSave(any())).thenReturn(mockItem);

        // when
        final ResultActions actions =
                mvc.perform(post("/api/v1/item")
                                .contentType(MediaType.APPLICATION_JSON)
                                .accept(MediaType.APPLICATION_JSON)
                                .characterEncoding("UTF-8")
                                .content("{" +
//                                        "\"idx\" : \"1\"," +
                                        "\"name\" : \"신상품1\"," +
                                        "\"description\" : \"신상품1 설명\"" +
                                        "}"));

        // then
        verify(itemProductService).itemSave(any());
        actions
                .andExpect(status().isCreated())
                .andExpect(jsonPath("name").value("신상품1"))
                .andExpect(jsonPath("description").value("신상품1 설명"))
                .andDo(print());
    }

    @Test
    void findAll() throws Exception {
        // given
        List<ProductDto> mockProducts = List.of(
                ProductDto.of().idx(1L).name("베베숲 물티슈").description("베베숲 물티슈 설명").build()
                ,ProductDto.of().idx(2L).name("여름 토퍼").description("여름 토퍼 설명").build()
                ,ProductDto.of().idx(3L).name("페이크 삭스").description("페이크 삭스 설명").build()
                ,ProductDto.of().idx(4L).name("우산").description("우산 설명").build()
        );
        when(itemProductService.findAll()).thenReturn(mockProducts);

        // when
        final ResultActions actions =
                mvc.perform(get("/api/v1/products"));

        // then
        String expectByUsername = "$.[?(@.username == '%s')]";
        String addressByCity = "$..address[?(@.city == '%s')]";


        actions
                .andExpect(status().isOk())
                .andExpect(jsonPath("$[0]").exists())
                .andExpect(jsonPath("$[1]").exists())
                .andExpect(jsonPath("$[2]").exists())
                .andExpect(jsonPath("$[3]").exists())
                .andExpect(jsonPath("$[0].name").value("베베숲 물티슈"))
                .andExpect(jsonPath("$[0].description").value("베베숲 물티슈 설명"))
                .andDo(print());
    }

    @Test
    void findById() throws Exception {
        // given
        Long idx = 1l;
        when(itemProductService.findByIdx(idx)).thenReturn(ProductDto.of().idx(1L).name("베베숲 물티슈").description("베베숲 물티슈 설명").build());

        // when
        final ResultActions actions =
                mvc.perform(get("/api/v1/products/{idx}", idx));

        // then
        actions
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.idx").value(1L))
                .andExpect(jsonPath("$.name").value("베베숲 물티슈"))
                .andExpect(jsonPath("$.description").value("베베숲 물티슈 설명"))
                .andDo(print());
    }

    @Test
    void insertProduct() throws Exception {
        // given
        when(itemProductService.insertProduct(any())).thenReturn(1);

        // when
        final ResultActions actions =
                mvc.perform(post("/api/v1/products")
                        .contentType(MediaType.APPLICATION_JSON)
                        .accept(MediaType.APPLICATION_JSON)
                        .characterEncoding("UTF-8")
                        .content("{" +
                                        "\"resultCount\" : 1" +
                                "}"));

        // then
        verify(itemProductService).insertProduct(any());
        actions
                .andExpect(status().isCreated())
                .andExpect(jsonPath("resultCount").value(1))
                .andDo(print());
    }

    @Test
    void updateProduct() throws Exception {
        // given
        when(itemProductService.updateProduct(any())).thenReturn(1);

        // when
        final ResultActions actions =
                mvc.perform(put("/api/v1/products")
                        .contentType(MediaType.APPLICATION_JSON)
                        .accept(MediaType.APPLICATION_JSON)
                        .characterEncoding("UTF-8")
                        .content("{" +
                                "\"resultCount\" : 1" +
                                "}"));

        // then
        verify(itemProductService).updateProduct(any());
        actions
                .andExpect(status().isOk())
                .andExpect(jsonPath("resultCount").value(1))
                .andDo(print());
    }

    @Test
    void deleteProduct() throws Exception {
        // given
        when(itemProductService.deleteProduct(any())).thenReturn(1);

        // when
        final ResultActions actions =
                mvc.perform(delete("/api/v1/products/{idx}", 1L)
                        .contentType(MediaType.APPLICATION_JSON)
                        .accept(MediaType.APPLICATION_JSON)
                        .characterEncoding("UTF-8")
                        .content("{" +
                                    "\"resultCount\" : 1" +
                                "}"));

        // then
        verify(itemProductService).deleteProduct(any());
        actions
                .andExpect(status().isOk())
                .andExpect(jsonPath("resultCount").value(1))
                .andDo(print());
    }
}
```


## 참고  
[Spring boot test, junit5, mockito 사용에 대한 정리](https://wan-blog.tistory.com/71)
[Junit5(Juptier) Service 테스트코드 작성법(Mock, MockBean 차이점 확인)](https://frozenpond.tistory.com/83)
[Mockito와 BDDMockito는 뭐가 다를까?](https://velog.io/@lxxjn0/Mockito%EC%99%80-BDDMockito%EB%8A%94-%EB%AD%90%EA%B0%80-%EB%8B%A4%EB%A5%BC%EA%B9%8C)
[[JUnit & Mockito] Verify Method Calls](https://velog.io/@dnjscksdn98/JUnit-Mockito-Verify-Method-Calls)

## Github
<https://github.com/sisipapa/junit5.git>  




