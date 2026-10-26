using GateApi.Services;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new Microsoft.OpenApi.Models.OpenApiInfo
    {
        Title = "Gate Validation API",
        Version = "v1",
        Description = "ASP.NET Core Web API для проверки кодов доступа (Акт I. Врата)."
    });
});

builder.Services.AddScoped<IAccessCodeValidationService, AccessCodeValidationService>();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI(options =>
{
    options.SwaggerEndpoint("/swagger/v1/swagger.json", "Gate Validation API v1");
    options.RoutePrefix = "swagger";
});

app.UseHttpsRedirection();
app.MapControllers();

app.Run();

namespace GateApi.Models
{
    public sealed class ValidateRequest
    {
        public string? AccessCode { get; set; }
    }
}

namespace GateApi.Models
{
    public sealed class ValidateResponse
    {
        public bool IsValid { get; init; }
        public IReadOnlyList<string> Errors { get; init; } = Array.Empty<string>();
        public string Message { get; init; } = string.Empty;
    }
}

namespace GateApi.Services
{
    using GateApi.Models;

    public interface IAccessCodeValidationService
    {
        ValidateResponse Validate(string? accessCode);
    }
}

namespace GateApi.Services
{
    using GateApi.Models;

    public sealed class AccessCodeValidationService : IAccessCodeValidationService
    {
        private const int MinLength = 8;
        private const int MaxLength = 16;

        private static readonly char[] AllowedSpecialChars = { '!', '@', '#', '$', '%' };

        public ValidateResponse Validate(string? accessCode)
        {
            var errors = new List<string>();

            if (string.IsNullOrWhiteSpace(accessCode))
            {
                errors.Add("Код доступа отсутствует или пуст.");
                return BuildResponse(errors);
            }

            var code = accessCode;

            if (code.Length < MinLength || code.Length > MaxLength)
            {
                errors.Add($"Длина кода должна быть от {MinLength} до {MaxLength} символов включительно.");
            }

            if (!code.Any(c => c >= 'A' && c <= 'Z'))
            {
                errors.Add("Нет заглавной латинской буквы.");
            }

            if (!code.Any(c => c >= 'a' && c <= 'z'))
            {
                errors.Add("Нет строчной латинской буквы.");
            }

            if (!code.Any(char.IsDigit))
            {
                errors.Add("Нет цифры.");
            }

            if (!code.Any(c => AllowedSpecialChars.Contains(c)))
            {
                errors.Add("Нет специального символа.");
            }

            if (code.Any(char.IsWhiteSpace))
            {
                errors.Add("Код не должен содержать пробелов.");
            }

            if (HasTripleIdenticalCharacters(code))
            {
                errors.Add("Обнаружены три одинаковых символа подряд.");
            }

            if (HasThreeOrMoreConsecutiveDigits(code))
            {
                errors.Add("Обнаружена последовательность из трёх или более подряд идущих цифр.");
            }

            if (!IsLatinLetter(code[0]))
            {
                errors.Add("Первый символ должен быть буквой.");
            }

            var lastChar = code[^1];
            if (!char.IsDigit(lastChar) && !AllowedSpecialChars.Contains(lastChar))
            {
                errors.Add("Последний символ должен быть цифрой или разрешённым специальным символом (!, @, #, $, %).");
            }

            return BuildResponse(errors);
        }

        private static ValidateResponse BuildResponse(List<string> errors)
        {
            if (errors.Count == 0)
            {
                return new ValidateResponse
                {
                    IsValid = true,
                    Errors = Array.Empty<string>(),
                    Message = "Код принят. Ворота открываются."
                };
            }

            return new ValidateResponse
            {
                IsValid = false,
                Errors = errors,
                Message = "Код отклонён."
            };
        }

        private static bool IsLatinLetter(char c) =>
            (c >= 'A' && c <= 'Z') || (c >= 'a' && c <= 'z');

        private static bool HasTripleIdenticalCharacters(string code)
        {
            if (code.Length < 3)
            {
                return false;
            }

            for (var i = 0; i <= code.Length - 3; i++)
            {
                if (code[i] == code[i + 1] && code[i + 1] == code[i + 2])
                {
                    return true;
                }
            }

            return false;
        }

        private static bool HasThreeOrMoreConsecutiveDigits(string code)
        {
            var consecutiveDigits = 0;

            foreach (var c in code)
            {
                if (char.IsDigit(c))
                {
                    consecutiveDigits++;
                    if (consecutiveDigits >= 3)
                    {
                        return true;
                    }
                }
                else
                {
                    consecutiveDigits = 0;
                }
            }

            return false;
        }
    }
}

namespace GateApi.Controllers
{
    using GateApi.Models;
    using GateApi.Services;
    using Microsoft.AspNetCore.Mvc;

    [ApiController]
    [Route("api/gates")]
    [Produces("application/json")]
    public sealed class GatesController : ControllerBase
    {
        private readonly IAccessCodeValidationService _validationService;

        public GatesController(IAccessCodeValidationService validationService)
        {
            _validationService = validationService;
        }

        [HttpPost("validate")]
        [ProducesResponseType(typeof(ValidateResponse), StatusCodes.Status200OK)]
        [ProducesResponseType(typeof(ValidateResponse), StatusCodes.Status400BadRequest)]
        public IActionResult Validate([FromBody] ValidateRequest? request)
        {
            if (request is null)
            {
                return BadRequest(new ValidateResponse
                {
                    IsValid = false,
                    Errors = new[] { "Тело запроса отсутствует или не является корректным JSON." },
                    Message = "Код отклонён."
                });
            }

            var result = _validationService.Validate(request.AccessCode);

            return result.IsValid
                ? Ok(result)
                : BadRequest(result);
        }
    }
}
