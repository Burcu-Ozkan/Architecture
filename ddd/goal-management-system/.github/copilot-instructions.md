# Copilot Instructions

## 1. Genel Mimari
- Katmanlar: Core (Domain + Abstraction), UseCases (Application), Infrastructure (Persistence), Web (Razor Pages UI), Tests (Unit / Integration / Functional)
- Domain mant��� Core i�inde; UseCases sadece orchestration yapar, EF Core eri�imini Infrastructure �stlenir.

### 1.1 Katman / Proje E�le�mesi
Katman | Uygulama Proje(leri) | Unit Test Proje(leri) | Notlar
-------|----------------------|------------------------|-------
Core | `GoalManager.Core` | (ayr� yok) | Domain modelleri, value objects, domain servisleri. Domain kurallar� �o�unlukla handler testleri ile dolayl� test edilir.
UseCases | `GoalManager.UseCases` | `GoalManager.UseCases.Tests` | Command/Query + Handler orchestration. �� kurallar� i�in birincil unit test hedefi.
Infrastructure | `GoalManager.Infrastructure` | (ayr� yok) | Repository & EF Core implementasyonlar�. Davran�� Integration Tests ile do�rulan�r.
Web (UI) | `GoalManager.Web` | (ayr� yok) | Razor Pages. UI do�rulamas� Functional (E2E) testlerle.
Tests (Integration) | `GoalManager.IntegrationTests` | - | Ger�ek (test) DB ve boundary etkile�imleri.
Tests (Functional) | `GoalManager.FunctionalTests` | - | HTTP pipeline & u�tan uca senaryolar.

---
## 2. Unit Test Kurallar�
1. Framework: xUnit.
2. Mock: NSubstitute (`Substitute.For<T>()`).
3. Test s�n�f ad�: `<SUTClassName>Tests` (�rn: `AddGoalPeriodCommandHandlerTests`).
4. Metot ad� pattern: `Handle_<Senaryo>_<BeklenenSonuc>` veya `Should_<Beklenen>_When_<Ko�ul>`.
5. Parametrik test: `[Theory]` + `[InlineData(...)]` yaln�z davran�� farkl�ysa. Ayn� sonucu tekrar eden varyasyon ekleme.
6. AAA D�zeni: Her test explicit ``// Arrange``, ``// Act``, ``// Assert`` bloklar� i�erir (gerekirse Arrange-Act veya Act-Assert birle�ik tek sat�r).
7. Guard, ba�ar� (happy path) ve her i� kural� ihlali ayr� test.
8. Tarih/saat ba��ml�l���: Provider mock => sabit de�er (`_time.Today().Returns(fixedDate);`).
9. Arg do�rulama: `Arg.Is<T>(x => x.Prop == val && x.Other > 0)`. Arg.Is kullan�m� tercih edilir, a��r� Arg.Any kullanma.
10. Gereksiz async kullanma; senkron davran�� i�in `Task.FromResult` yeterli.
11. Yeni `.cs` dosyalar� UTF-8.
12. Yeni eklenen i� kural� / branch test edilmeli (gereksiz %100 kovalanmaz; kritik dallar kapsan�r).
13. T�m testler build pipeline'da �al���p ge�meli.
14. Test yaz�lan kod %100 kapsanmal� (kodda cover olmayan sat�r varsa test ekle).
15. Category i�in: `[Trait("Category", "GoalManagement/AddGoalPeriod")]` format�n� kullan.
16. Hata mesaj� assertion: Tam e�le�me gerekirse `Assert.Contains(result.Errors, e => e == "..." )`; par�al� i�in `e.Contains("fragment", StringComparison.OrdinalIgnoreCase)`.

### 2.1 �rnek
```csharp
[Fact]
public async Task Handle_Returns_error_when_period_already_exists()
{
  // Arrange
  var repo = Substitute.For<IRepository<GoalPeriod>>();
  repo.AnyAsync(Arg.Any<GoalPeriodByTeamIdAndYearSpec>(), Arg.Any<CancellationToken>())
      .Returns(true);
  var sut = new AddGoalPeriodCommandHandler(repo);
  var cmd = new AddGoalPeriodCommand(TeamId: 10, UserId: 1, Year: 2025);

  // Act
  var result = await sut.Handle(cmd, CancellationToken.None);

  // Assert
  Assert.False(result.IsSuccess);
  Assert.Contains(result.Errors, e => e.Contains("already exists", StringComparison.OrdinalIgnoreCase));
  await repo.DidNotReceive().AddAsync(Arg.Any<GoalPeriod>(), Arg.Any<CancellationToken>());
}
```

### 2.2 Parametrik (Theory) �rne�i
```csharp
[Theory]
[InlineData(0)]
[InlineData(-1)]
public async Task Handle_Returns_invalid_for_non_positive_year(int invalidYear)
{
  // Arrange
  var repo = Substitute.For<IRepository<GoalPeriod>>();
  var sut = new AddGoalPeriodCommandHandler(repo);
  var cmd = new AddGoalPeriodCommand(TeamId: 1, UserId: 1, Year: invalidYear);

  // Act
  var result = await sut.Handle(cmd, CancellationToken.None);

  // Assert
  Assert.False(result.IsSuccess);
  Assert.Contains(result.Errors, e => e.Contains("year", StringComparison.OrdinalIgnoreCase));
}
```

### 2.3 Guard Test �rne�i
```csharp
[Fact]
public void Ctor_Throws_ArgumentNullException_when_repo_null()
{
  // Arrange / Act & Assert
  Assert.Throws<ArgumentNullException>(() => new AddGoalPeriodCommandHandler(null!));
}
```

---
## 3. UseCases Katman� Factory �rne�i
Basit handler factory gerekirse (tekrar eden setup a��rla��rsa) test s�n�f� i�inde tutulur.
```csharp
private static AddGoalPeriodCommandHandler CreateHandler(IRepository<GoalPeriod>? repo = null)
  => new(repo ?? Substitute.For<IRepository<GoalPeriod>>());
```
Gereksiz abstraksiyon ekleme; do�rudan inline kullanmak tercih edilir.

---
## 4. Yeni Test Yaz�m Ak��� (Checklist)
1. SUT public API & ba��ml�l�klar incelendi
2. Gerekli mock'lar olu�turuldu
3. Guard senaryolar� yaz�ld�
4. En az bir ba�ar�l� senaryo
5. Her i� kural� ihlali ayr� test
6. Kenar de�er(ler) (min / max / 0 / negatif) test edildi
7. Mock Received / DidNotReceive do�rulamalar� eklendi
8. Hata mesaj� assertion uygun
9. Gereksiz parametrik test yok
10. Paralel �al��may� bozan payla��lan mutable static yok
11. Build + t�m testler ge�ti

---
## 5. Kenar Durum �rnekleri (K�sa Rehber)
- Say�sal: 0, 1, negatif, maksimum (int.MaxValue)
- Koleksiyon: bo�, tek eleman, limit �st�
- Tarih: y�l sonu, ay sonu, 29 �ubat
- Concurrency: �lk deneme �ak��ma (�rn. �zel exception), ikinci ba�ar�
- Idempotent: Ayn� komut ikinci kez => ek yan etki olmamal�
- Cache/Lookup: bulunamad� -> eklendi -> tekrar �a�r� vs.

Mini concurrency pattern:
```csharp
var call = 0;
repo.AnyAsync(...).Returns(_ => ++call == 1 ? throw new ConcurrencyException() : false);
```

---
## 6. NSubstitute �pu�lar�
- �a�r� say�s�: `await repo.Received(1).AddAsync(Arg.Any<GoalPeriod>(), Arg.Any<CancellationToken>());`
- Parametre do�rulama: `Arg.Is<GoalPeriod>(p => p.TeamId == cmd.TeamId && p.Year == cmd.Year)`
- Guard sonras�: `repo.DidNotReceive().AddAsync(Arg.Any<GoalPeriod>(), Arg.Any<CancellationToken>());`
- S�ra (gerekirse):
```csharp
Received.InOrder(() => {
  repo.Received().AnyAsync(Arg.Any<GoalPeriodByTeamIdAndYearSpec>(), Arg.Any<CancellationToken>());
  repo.Received().AddAsync(Arg.Any<GoalPeriod>(), Arg.Any<CancellationToken>());
});
```

Yayg�n hatalar: Act yap�lmadan Received �a��rmak, a��r� `Arg.Any`, test i�inde mock davran��� sonradan de�i�tirmek, tek testte birden fazla i� kural� do�rulamak.

---
## 7. Result -> HTTP (Razor Pages) Mapping (Referans)
Status | HTTP | Not
-------|------|----
Success | 200/204 | Value varsa 200, yoksa 204
NotFound | 404 | Kay�t yok
Invalid | 400 | Validation / i� kural� ihlali
Conflict | 409 | �ak��ma / concurrency
Unauthorized | 401 | Kimlik do�rulama yok
Forbidden | 403 | Yetki yok
Error | 500 | Log + generic mesaj

Basit handler kullan�m �rne�i (PageModel OnPost):
```csharp
var result = await _mediator.Send(command);
return result.Status switch
{
  ResultStatus.Success    => RedirectToPage("Detail", new { id = result.Value }),
  ResultStatus.NotFound   => NotFound(),
  ResultStatus.Invalid    => BadRequest(result.Errors),
  ResultStatus.Conflict   => StatusCode(409, result.Errors),
  _                       => StatusCode(500)
};
```

---